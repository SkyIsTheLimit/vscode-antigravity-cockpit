# How Antigravity Usage Detection Works

This document explains the technical mechanisms used by Antigravity Cockpit to detect and monitor Google Antigravity AI usage.

## Table of Contents

1. [Overview](#overview)
2. [Detection Architecture](#detection-architecture)
3. [Local Process Detection](#local-process-detection)
4. [Authorized Account Monitoring](#authorized-account-monitoring)
5. [Platform-Specific Implementation](#platform-specific-implementation)
6. [Test Coverage](#test-coverage)

---

## Overview

Antigravity Cockpit uses a **two-tier detection system** to monitor Google Antigravity AI model quota usage:

1. **Local Process Detection**: Scans running system processes to find the Antigravity Language Server
2. **Authorized Account Monitoring**: Uses OAuth credentials to fetch quota data directly from configured accounts

Users can switch between these modes using the `agCockpit.quotaSource` configuration setting (`local` or `authorized`).

---

## Detection Architecture

### Core Components

| Component | File | Purpose |
|-----------|------|---------|
| **ProcessHunter** | `/src/engine/hunter.ts` | Discovers Antigravity processes and extracts connection parameters |
| **ReactorCore** | `/src/engine/reactor.ts` | Communicates with Antigravity API and manages authorized accounts |
| **Detection Strategies** | `/src/engine/strategies.ts` | Platform-specific process scanning logic |
| **CockpitToolsLocal** | `/src/services/cockpitToolsLocal.ts` | Reads local account configuration files |

### Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│                  Antigravity Cockpit                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌────────────────────┐         ┌─────────────────────┐    │
│  │  ProcessHunter     │         │   ReactorCore       │    │
│  │  (Local Detection) │         │ (Authorized Accts)  │    │
│  └────────┬───────────┘         └──────────┬──────────┘    │
│           │                                 │               │
│           ▼                                 ▼               │
│  ┌────────────────────┐         ┌─────────────────────┐    │
│  │  Platform Strategy │         │ OAuth Credentials   │    │
│  │  (Win/Mac/Linux)   │         │   Storage           │    │
│  └────────┬───────────┘         └──────────┬──────────┘    │
└───────────┼──────────────────────────────────┼──────────────┘
            │                                  │
            ▼                                  ▼
   ┌────────────────┐                ┌──────────────────┐
   │ System Process │                │  Antigravity API │
   │     List       │                │   (Remote)       │
   └────────────────┘                └──────────────────┘
```

---

## Local Process Detection

### How It Works

The `ProcessHunter` class implements a multi-stage scanning process to locate the Antigravity Language Server running on the local machine.

### Detection Criteria

A process is identified as Antigravity if it contains **all three** of these command-line parameters:

1. `--extension_server_port <port>` - The port for VS Code extension communication
2. `--csrf_token <token>` - Security token for API requests
3. `--app_data_dir antigravity` - Identifies this as an Antigravity process (most reliable identifier)

### Three-Stage Scanning Process

#### Stage 1: Process Name Scan

**Purpose**: Find candidate processes by searching for the Antigravity Language Server executable

**Platform-Specific Commands**:

- **Windows**: 
  ```powershell
  Get-CimInstance Win32_Process | Where-Object { $_.CommandLine -match 'antigravity' } | Select-Object ProcessId,CommandLine
  ```

- **macOS/Linux**:
  ```bash
  ps -ww -eo pid,ppid,args | grep -i antigravity
  ```

**Retry Logic**: Up to 3 attempts with configurable timeout
- Windows includes special handling for PowerShell cold-start delays
- Falls back to keyword search if process name scan fails

#### Stage 2: Port Identification

**Purpose**: Extract the HTTP server port from the identified process

**Platform-Specific Commands**:

- **Windows**:
  ```powershell
  Get-NetTCPConnection -OwningProcess <PID> | Where-Object { $_.State -eq 'Listen' }
  ```

- **macOS/Linux** (priority order):
  1. `lsof -Pan -p <PID> -iTCP -sTCP:LISTEN` (preferred - most reliable)
  2. `ss -tlnp | grep <PID>` (fallback)
  3. `netstat -tlnp | grep <PID>` (last resort)

**Logic**: Filters for LISTEN state TCP connections on localhost

#### Stage 3: Connection Verification

**Purpose**: Verify the port is actually an Antigravity server by making an API request

**Process**:
1. Extract CSRF token from process command line
2. Send HTTPS GET request to `https://127.0.0.1:<port>/api/v1/unleash-data`
3. Include extracted CSRF token in headers
4. Validate response contains expected data structure

**Success Criteria**: 
- Response status 200
- Valid JSON response
- Contains expected Antigravity data fields

### Error Handling

The detection system includes comprehensive error recovery:

- **PowerShell Delays**: Automatic retries for Windows PowerShell startup delays (up to +3 seconds without consuming retry count)
- **Command Failures**: Falls back to alternative commands (e.g., `lsof` → `ss` → `netstat`)
- **Keyword Search**: If process name search fails, scans all processes for `csrf_token` keyword
- **Detailed Diagnostics**: Logs scan method, attempts, candidates found, and ports tested for troubleshooting

### Result Structure

```typescript
interface EnvironmentScanResult {
  extensionPort: number;  // Port for VS Code extension API
  connectPort: number;    // Port for HTTP/HTTPS connections
  csrfToken: string;      // Security token for authenticated requests
}
```

---

## Authorized Account Monitoring

### How It Works

The `ReactorCore` class manages monitoring through authorized OAuth accounts, bypassing the need for a local Antigravity process.

### Account Sources

1. **OAuth Credentials**: Stored securely in VS Code Secret Storage
   - Managed by `credentialStorage` service
   - Encrypted at rest
   - Used for remote API authentication

2. **Local Account Index**: Read from `~/.antigravity_cockpit/accounts.json`
   - Lists available accounts
   - Provides account metadata
   - Syncs with local Antigravity installation

### Monitoring Process

1. **Account Selection**: User selects active account via UI
2. **Credential Retrieval**: OAuth tokens fetched from secure storage
3. **API Communication**: HTTPS requests to Antigravity API endpoints
4. **Quota Caching**: Responses cached in `quota_api_cache` for performance
5. **Auto-Refresh**: Periodic updates based on `agCockpit.refreshInterval` setting

### Account Switching

The system handles mid-fetch account switching:
- Detects active account changes
- Cancels in-progress requests
- Automatically retries with new account
- Updates UI to reflect current account

### Source Identification

Quota data includes a `source` field:
- `'local'`: Data from local process detection
- `'authorized'`: Data from authorized account API

---

## Platform-Specific Implementation

### Windows Strategy

**File**: `/src/engine/strategies.ts` - `WindowsStrategy` class

**Process Scanning**:
- Uses PowerShell CIM cmdlets for process enumeration
- Includes full command line in results
- Special handling for PowerShell startup delays

**Port Discovery**:
- Uses `Get-NetTCPConnection` for reliable port identification
- Filters by process ID and LISTEN state
- Supports both IPv4 and IPv6

**Optimizations**:
- Retry logic accounts for PowerShell initialization time
- Keyword fallback search for edge cases
- Detailed error reporting for troubleshooting

### Unix Strategy (macOS/Linux)

**File**: `/src/engine/strategies.ts` - `UnixStrategy` class

**Process Scanning**:
- Uses standard `ps` command with wide output format
- Searches entire command line arguments
- Case-insensitive matching

**Port Discovery (Priority Order)**:
1. **lsof** (preferred):
   ```bash
   lsof -Pan -p <PID> -iTCP -sTCP:LISTEN
   ```
   - Most reliable on macOS
   - Works on most Linux distributions
   - Provides complete connection information

2. **ss** (fallback):
   ```bash
   ss -tlnp | grep <PID>
   ```
   - Modern alternative to netstat
   - Fast and efficient
   - Available on most recent Linux systems

3. **netstat** (last resort):
   ```bash
   netstat -tlnp | grep <PID>
   ```
   - Legacy tool, widely available
   - Works on older systems
   - Less detailed output

**Error Handling**:
- Graceful fallback between tools
- Detailed logging of which tool was used
- Reports tool availability for diagnostics

---

## Test Coverage

### Test File

**Location**: `/src/engine/strategies.test.ts`

### Test Cases

#### 1. Valid Process Detection

Tests that processes with all required parameters are correctly identified:

```typescript
// Windows format
'--extension_server_port 53125 --csrf_token abc-123 --app_data_dir antigravity'

// Unix format with equals
'--extension_server_port=53125 --csrf_token=abc-123 --app_data_dir=antigravity'
```

**Expected**: ✅ Process identified as Antigravity

#### 2. Missing Parameter Detection

Tests rejection of processes missing required parameters:

```typescript
// Missing extension port
'--csrf_token abc-123 --app_data_dir antigravity'

// Missing CSRF token  
'--extension_server_port 53125 --app_data_dir antigravity'

// Missing app_data_dir
'--extension_server_port 53125 --csrf_token abc-123'
```

**Expected**: ❌ Process rejected (not Antigravity)

#### 3. Wrong app_data_dir Value

Tests that only `antigravity` value is accepted:

```typescript
'--extension_server_port 53125 --csrf_token abc-123 --app_data_dir other_app'
```

**Expected**: ❌ Process rejected (wrong application)

#### 4. Parameter Format Variations

Tests both space and equals-separated parameter formats:

```typescript
// Space-separated (Windows style)
'--param value'

// Equals-separated (Unix style)
'--param=value'
```

**Expected**: ✅ Both formats supported

### Running Tests

```bash
# Run all tests
npm test

# Run only detection strategy tests
npm test -- strategies.test.ts
```

---

## Configuration

### Quota Source Selection

Set in VS Code settings:

```json
{
  "agCockpit.quotaSource": "local"  // or "authorized"
}
```

- **`local`**: Uses process detection (requires Antigravity running locally)
- **`authorized`**: Uses OAuth credentials (works remotely)

### Display Modes

Both detection methods work with both display modes:

```json
{
  "agCockpit.displayMode": "webview"  // or "quickpick"
}
```

### Refresh Interval

Controls how often quota data is updated:

```json
{
  "agCockpit.refreshInterval": 120  // seconds (10-3600)
}
```

---

## Troubleshooting

### No Process Found

**Symptom**: "Systems Offline" message in dashboard

**Possible Causes**:
1. Antigravity Language Server not running
2. Process detection permissions issue
3. Antivirus blocking process enumeration

**Solutions**:
1. Start Antigravity application
2. Check VS Code has permission to list processes
3. Switch to "authorized" mode: `"agCockpit.quotaSource": "authorized"`

### Port Detection Fails

**Symptom**: Process found but connection fails

**Possible Causes**:
1. Missing `lsof`/`ss`/`netstat` tools on Linux
2. Firewall blocking localhost connections
3. Wrong port extracted from process

**Solutions**:
1. Install required tools: `sudo apt-get install lsof` or `ss`
2. Check firewall settings for localhost
3. View logs: "Antigravity Cockpit: Show Logs" command

### Authorized Mode Not Working

**Symptom**: No data in authorized mode

**Possible Causes**:
1. No OAuth credentials configured
2. Credentials expired or invalid
3. Network connectivity issues

**Solutions**:
1. Configure account in Cockpit Tools
2. Re-authorize account
3. Check network connection and firewall

---

## Security Considerations

### Local Mode
- No credentials stored
- Only reads from local system processes
- CSRF tokens used for localhost API security
- All communication over localhost HTTPS

### Authorized Mode
- OAuth tokens stored in VS Code Secret Storage (encrypted)
- Credentials never logged or exposed
- HTTPS for all API communication
- Automatic token refresh handling

---

## Performance Optimization

### Local Detection
- Process scan cached for short duration
- Parallel port testing when multiple candidates found
- Platform-specific optimizations (e.g., PowerShell warm-up handling)

### Authorized Mode
- API responses cached in `quota_api_cache`
- Configurable refresh intervals
- Efficient account switching without full re-initialization

---

## Summary

Antigravity Cockpit provides robust detection of Antigravity AI usage through:

✅ **Multi-platform support**: Windows, macOS, Linux  
✅ **Dual detection modes**: Local process and authorized accounts  
✅ **Comprehensive validation**: Three-stage verification process  
✅ **Error recovery**: Automatic fallbacks and retries  
✅ **Security**: Encrypted credential storage, HTTPS communication  
✅ **Performance**: Intelligent caching and optimization  
✅ **Test coverage**: Automated tests for core detection logic

For additional help, see:
- [README.md](../README.md) - General usage guide
- [README.en.md](../README.en.md) - English documentation
- GitHub Issues: [Report a problem](https://github.com/jlcodes99/vscode-antigravity-cockpit/issues)
