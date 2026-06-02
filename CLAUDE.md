# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Power BI Gateway Monitoring Solution** - A centralized system for collecting, storing, and analyzing Power BI On-Premises Data Gateway logs from multiple gateway clusters. Logs are uploaded to Azure Data Lake Storage Gen2 (ADLS Gen2) and visualized via Power BI reports.

**Status**: Deprecated - repository no longer actively maintained. Recommended migration: [Fabric Platform Monitoring](https://github.com/microsoft/fabric-toolbox/tree/main/monitoring/fabric-platform-monitoring)

## Architecture

### Core Components

| Component | Purpose |
|-----------|---------|
| **Run.ps1** | Entry point orchestrator. Loads Config.json, sets location, and invokes scripts |
| **UploadGatewayLogs.ps1** | Main worker script. Discovers gateway logs, processes them by date, uploads to ADLS Gen2, maintains state.json for incremental runs |
| **Utils.psm1** | PowerShell module with Azure Storage helpers: `Add-FileToBlobStorage`, `Add-FolderToBlobStorage`, `Get-ArrayInBatches` |
| **Config.json** | Configuration template. Define: GatewayLogsPath, StorageAccountConnStr, StorageAccountContainerName, StorageAccountContainerRootPath, OutputPath |
| **state.json** | Persists execution state (LastRun timestamp for GatewayLogs). Enables incremental log processing. Auto-created on first run |
| **PBIP/** | Power BI Integrated Project with Dataset and Report definitions (newer format replacing .pbix) |
| **.pbit files** | Power BI templates (FromLake.pbit for ADLS data, FromDisk.pbit for local logs). Parameterized for easy deployment |

### Data Flow

```
Gateway Server
  ↓ (*.log files, Report_*.log, ConfigurationProperties.json)
UploadGatewayLogs.ps1
  ↓ (reads from GatewayLogsPath, filters by LastRun from state.json)
ProcessLogFiles function
  ↓ (stages in OutputPath, parses date from filename, organizes by gatewayid/yyyy/MM/dd)
Azure ADLS Gen2 (via Utils.psm1)
  ↓ (StorageAccountContainerRootPath/{gatewayid}/{logs,metadata,reports}/)
Power BI Template (.pbit)
  ↓ (connects via DFS endpoint: https://storage.dfs.core.windows.net/)
Power BI Report
```

### Key Processing Details

- **Incremental uploads**: State file tracks last execution time. Only new/modified logs since LastRun are processed.
- **Date parsing**: Filenames like `gatewayinfo20240602.001` → extracted to yyyy/MM/dd storage path.
- **Metadata discovery**: Attempts to infer GatewayId from Report_*.log files (GatewayObjectId CSV field) if not manually provided in Config.json.
- **Server info**: Collects NumberOfCores from ComputerInfo if not pre-configured.
- **Blob naming**: Files stored as `{StorageAccountContainerRootPath}/{gatewayid}/logs|metadata|reports/{date}/{filename}`.

## Common Development Tasks

### Running the Main Script

```powershell
# Default - uses .\Config.json
.\Run.ps1

# Custom config path
.\Run.ps1 -configFilePath ".\Config-Production.json"

# Requires PowerShell 7+
pwsh .\Run.ps1
```

### Testing a Single Script

```powershell
# Load Utils module and run UploadGatewayLogs directly
Import-Module ".\Utils.psm1" -Force
$config = Get-Content ".\Config.json" | ConvertFrom-Json
& ".\UploadGatewayLogs.ps1" -config $config -stateFilePath ".\state-test.json"
```

### Validating Config Before Deployment

```powershell
$config = Get-Content ".\Config.json" | ConvertFrom-Json
Write-Host "Gateway Path: $($config.GatewayLogsPath)" 
Write-Host "Storage Container: $($config.StorageAccountContainerName)"
Write-Host "Output Path: $($config.OutputPath)"
```

### Scheduling on Windows (Task Scheduler)

Use `Run.cmd` wrapper or create a task:
- Trigger: Daily or hourly
- Action: `pwsh -NoProfile -ExecutionPolicy Bypass -File "C:\Path\To\Run.ps1"`
- Run with: Service account that has read access to gateway logs and write access to ADLS Gen2

## Configuration

### Config.json Fields

| Field | Type | Description |
|-------|------|-------------|
| `GatewayLogsPath` | string or array | Path(s) to gateway log directories. Can be string (auto-discover) or array of objects with manual metadata (Path, GatewayId, GatewayName, GatewayCluster, Server, Version, NumberOfCores) |
| `StorageAccountConnStr` | string | Azure Storage connection string for ADLS Gen2 account |
| `StorageAccountContainerName` | string | Container name (e.g., "pbigatewaymonitor") |
| `StorageAccountContainerRootPath` | string | Root path within container (e.g., "raw") |
| `OutputPath` | string | Local staging directory for temp files before upload (default: ".\Data") |

### Example Config with Manual Metadata

```json
{
  "GatewayLogsPath": [
    {
      "Path": "C:\\Windows\\ServiceProfiles\\PBIEgwService\\AppData\\Local\\Microsoft\\On-premises data gateway",
      "GatewayId": "12345678-1234-1234-1234-123456789012",
      "GatewayName": "Production Gateway 1",
      "GatewayCluster": "Cluster-A",
      "Server": "GatewayServer01",
      "NumberOfCores": 8
    }
  ],
  "StorageAccountConnStr": "[ADLS Gen 2 Connection String]",
  "StorageAccountContainerName": "pbigatewaymonitor",
  "StorageAccountContainerRootPath": "raw",
  "OutputPath": ".\\Data"
}
```

## Secrets & Azure Authentication

### Sensitive Data (in .gitignore)

- `Config - *.json` - Never commit configs with real connection strings
- `ServicePrincipal.json` - Service principal credentials
- `GraphAppSecret.txt` - Graph API secrets
- `pbitoken*.bin` - Power BI authentication tokens
- `devicecodetoken.bin` - Device code tokens

### Authentication

Scripts use Azure PowerShell modules (Az.Accounts, Az.Storage). Authentication is handled via:
- Connection string from Config.json (storage account key embedded)
- Service Principal (if using certificate-based auth instead)
- Device code flow for interactive sessions

### Rotation

If Config.json (with connection string) is ever committed accidentally:
1. Rotate the storage account key immediately in Azure Portal
2. Update all Config.json files with new connection string
3. Review deployment targets for stale credentials

## Power BI Templates (.pbit)

### FromLake.pbit
Connects to ADLS Gen2 data via DFS endpoint. Parameterized for:
- **DataLocation**: DFS path, e.g., `https://storage.dfs.core.windows.net/pbigatewaymonitor/raw`
- **NumberDays**: Filter to recent N days (null = all)
- **SinceDate**: Filter from specific date (overrides NumberDays)
- **MaxLogTextLength**: Truncate log text column (default 1000)
- **LogFilters**: Comma-separated log types (e.g., "gatewayerrors,gatewayinfo")
- **GatewayFilters**: Comma-separated gateway IDs (null = all)

### FromDisk.pbit
Connects to local gateway log files. Does not require ADLS Gen2 or this repo's PowerShell infrastructure. Useful for:
- Quick ad-hoc log analysis on gateway servers
- Pre-deployment validation
- Standalone gateway log exploration

## PBIP Project Structure

Located in `./PBIP/`:
- `Gateway Monitor.Dataset/` - Semantic model with tables, measures, relationships
- `Gateway Monitor.Report/` - Report pages and visualizations
- `Gateway Monitor.pbip` - Project metadata

Uses Power BI's new integrated project format (`.pbip` with folder structure instead of monolithic `.pbix`). Suitable for source control via git with proper merge handling.

## Important Notes

### On Gateway Log File Patterns

The script discovers and processes:
- `*.log` files in gateway directory root
- `*Report_*.log` files (recursive search) - contains GatewayObjectId
- `*ConfigurationProperties.json` files - gateway configuration snapshots

Parsing logic depends on standard gateway log naming (e.g., `gatewayinfo20240602.001`, `gatewayerrors20240602.002`).

### On Date Extraction

If filename contains `YYYYMMDD` pattern, that date is used for storage path organization. Otherwise, execution date is used. This enables backdating of logs if discovered later.

### On Incremental Processing

Logs are filtered by `LastWriteTimeUtc > state.GatewayLogs.LastRun`. State file persists per execution to avoid re-uploading. Deleting state.json forces full re-upload on next run (useful for recovery/backfill).

### On File Locking

Script stages logs in OutputPath first (local copy) before uploading to blob storage. This is necessary because Windows may lock active log files. After successful blob upload, local copies are deleted to save disk space.

## Debugging

### Enable Verbose Output

```powershell
$VerbosePreference = "Continue"
.\Run.ps1 -Verbose
```

### Trace Execution

Add debug breakpoints in PowerShell ISE, or use:
```powershell
Set-PSDebug -Trace 1  # 0 to disable
```

### Check State File

```powershell
Get-Content ".\state.json" | ConvertFrom-Json | ConvertTo-Json -Depth 10
```

### Validate Azure Connection

```powershell
$ctx = New-AzStorageContext -ConnectionString $config.StorageAccountConnStr
Get-AzStorageContainer -Context $ctx
```

## Testing

No automated test suite exists. Manual validation approach:

1. **Config validation**: Ensure paths exist, connection string is valid
2. **Dry run**: Run script against test gateway logs, verify Output staging
3. **Blob validation**: Connect to storage, verify files appear with correct structure
4. **Power BI test**: Open .pbit template, point to test data location, verify reports load

### Test Data

Copy a few days of real logs to a test folder, update Config.json to point there, and run.

## Related Documentation

- **README.md** - User-facing setup instructions, architecture diagram, template parameters, report page descriptions
- **LinkedInBlog**: https://www.linkedin.com/pulse/power-bi-gateway-monitoring-troubleshooting-solution-rui-romano/
- **Microsoft Gateway Docs**: https://docs.microsoft.com/en-us/data-integration/gateway/service-gateway-log-files
- **Azure ADLS Gen2**: https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction
