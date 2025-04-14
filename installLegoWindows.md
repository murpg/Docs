# Installing Lego ACME Client for Windows

This guide provides instructions for installing the [Lego ACME client](https://github.com/go-acme/lego) alongside CommandBox on Windows systems.

## Overview

Lego is a Let's Encrypt client and ACME library written in Go. It can help you automate the process of obtaining, renewing, and using SSL certificates through the ACME protocol.

This installation script:
- Automatically detects and finds your CommandBox installation
- Downloads the latest version of Lego from GitHub
- Extracts the Lego executable to your CommandBox directory
- Adds the directory to your PATH (optional)
- Provides troubleshooting tips for common DNS providers

## Prerequisites

- Windows with PowerShell 5.1 or later
- CommandBox already installed on your system
- Administrator rights (optional, required only for Machine-wide PATH changes)

## Installation Script

Copy the entire script below and save it as `Install-Lego.ps1`, then run it in PowerShell.

```powershell
# Install-Lego.ps1
# Script to download and install the Lego ACME client alongside CommandBox
# Requires PowerShell 5.1 or later

# Configuration Variables
$legoExeName = "lego.exe"

function Get-LatestLegoVersion {
    Write-Host "Fetching latest Lego release information..." -ForegroundColor Cyan
    
    try {
        # GitHub API URL for Lego releases
        $apiUrl = "https://api.github.com/repos/go-acme/lego/releases/latest"
        
        # Invoke REST method to get the latest release info
        $response = Invoke-RestMethod -Uri $apiUrl -Method Get -ErrorAction Stop
        
        # Extract the version tag (should be in format "vX.Y.Z")
        $latestVersion = $response.tag_name
        
        if ([string]::IsNullOrEmpty($latestVersion)) {
            throw "Could not find version tag in GitHub API response"
        }
        
        Write-Host "Found latest version: $latestVersion" -ForegroundColor Cyan
        
        # Get the download URL for Windows amd64 zip
        $asset = $null
        foreach ($assetItem in $response.assets) {
            if ($assetItem.name -like "*windows_amd64.zip") {
                $asset = $assetItem
                break
            }
        }
        
        if ($null -eq $asset) {
            throw "Could not find Windows AMD64 asset in release assets"
        }
        
        $downloadUrl = $asset.browser_download_url
        
        if ([string]::IsNullOrEmpty($downloadUrl)) {
            throw "Could not extract download URL from asset"
        }
        
        Write-Host "Found download URL: $downloadUrl" -ForegroundColor Cyan
        
        return @{
            Version = $latestVersion
            DownloadUrl = $downloadUrl
        }
    }
    catch {
        Write-Warning "Failed to fetch latest version: $_"
        Write-Host "Falling back to default version v4.22.2" -ForegroundColor Yellow
        
        # Fallback to known version if GitHub API fails
        return @{
            Version = "v4.22.2"
            DownloadUrl = "https://github.com/go-acme/lego/releases/download/v4.22.2/lego_v4.22.2_windows_amd64.zip"
        }
    }
}

# Get latest Lego version info
$legoInfo = Get-LatestLegoVersion
$legoVersion = $legoInfo.Version
$legoDownloadUrl = $legoInfo.DownloadUrl
$tempZipFile = Join-Path $env:TEMP "lego_$($legoVersion)_windows_amd64.zip"

Write-Host "Using Lego version: $legoVersion" -ForegroundColor Green

function Find-BoxBinary {
    # First check if 'box' is in PATH
    $boxCmd = Get-Command -Name "box" -ErrorAction SilentlyContinue
    
    if ($boxCmd) {
        return $boxCmd.Source
    }
    
    # Common locations to check for box binary
    $commonLocations = @(
        "${env:ProgramFiles}\CommandBox",
        "${env:ProgramFiles(x86)}\CommandBox",
        "${env:USERPROFILE}\CommandBox",
        "${env:USERPROFILE}\.CommandBox"
    )
    
    foreach ($location in $commonLocations) {
        $potentialPath = Join-Path $location "box.exe"
        if (Test-Path $potentialPath) {
            return $potentialPath
        }
    }
    
    return $null
}

function Add-ToPath {
    param (
        [string]$PathToAdd,
        [ValidateSet("User", "Machine")]
        [string]$Scope = "User"
    )
    
    # Get current path
    if ($Scope -eq "User") {
        $currentPath = [Environment]::GetEnvironmentVariable("Path", [EnvironmentVariableTarget]::User)
    } else {
        # Requires admin rights
        $currentPath = [Environment]::GetEnvironmentVariable("Path", [EnvironmentVariableTarget]::Machine)
    }
    
    # Check if path already exists in PATH
    if ($currentPath -split ";" -contains $PathToAdd) {
        Write-Host "Path already exists in $Scope PATH: $PathToAdd" -ForegroundColor Yellow
        return
    }
    
    # Add path to PATH
    $newPath = "$currentPath;$PathToAdd"
    
    try {
        [Environment]::SetEnvironmentVariable("Path", $newPath, [EnvironmentVariableTarget]::$Scope)
        Write-Host "Added to $Scope PATH: $PathToAdd" -ForegroundColor Green
        
        # Also add to current process PATH
        $env:Path = "$env:Path;$PathToAdd"
    } catch {
        Write-Error "Failed to update PATH. If adding to Machine PATH, ensure you run as administrator."
        Write-Error $_.Exception.Message
    }
}

function Test-Admin {
    $currentUser = New-Object Security.Principal.WindowsPrincipal([Security.Principal.WindowsIdentity]::GetCurrent())
    return $currentUser.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
}

# Main script execution starts here
Write-Host "Starting Lego ACME client installation for CommandBox..." -ForegroundColor Cyan

# Step 1: Find the box binary
$boxPath = Find-BoxBinary
if (-not $boxPath) {
    Write-Error "ERROR: CommandBox binary (box.exe) not found. Please install CommandBox first."
    exit 1
}

$boxDirectory = Split-Path -Parent $boxPath
Write-Host "Found CommandBox at: $boxPath" -ForegroundColor Green

# Step 2: Download Lego binary
Write-Host "Downloading Lego $legoVersion from $legoDownloadUrl..." -ForegroundColor Cyan
try {
    Invoke-WebRequest -Uri $legoDownloadUrl -OutFile $tempZipFile
} catch {
    Write-Error "Failed to download Lego binary: $_"
    exit 1
}

if (-not (Test-Path $tempZipFile)) {
    Write-Error "Download failed. ZIP file not found at: $tempZipFile"
    exit 1
}

# Step 3: Extract Lego binary to Box directory
$legoDestination = Join-Path $boxDirectory $legoExeName
Write-Host "Extracting Lego to: $legoDestination" -ForegroundColor Cyan

try {
    Add-Type -AssemblyName System.IO.Compression.FileSystem
    $zip = [System.IO.Compression.ZipFile]::OpenRead($tempZipFile)
    $legoEntry = $zip.Entries | Where-Object { $_.Name -eq $legoExeName }
    
    if (-not $legoEntry) {
        Write-Error "Lego executable not found in the downloaded zip file."
        $zip.Dispose()
        exit 1
    }
    
    # Extract the Lego executable
    [System.IO.Compression.ZipFileExtensions]::ExtractToFile($legoEntry, $legoDestination, $true)
    $zip.Dispose()
} catch {
    Write-Error "Failed to extract Lego: $_"
    exit 1
} finally {
    if ($zip) { $zip.Dispose() }
    # Clean up the temporary zip file
    Remove-Item $tempZipFile -Force -ErrorAction SilentlyContinue
}

# Step 4: Verify Lego extraction was successful
if (-not (Test-Path $legoDestination)) {
    Write-Error "Extraction failed. Lego executable not found at: $legoDestination"
    exit 1
}

Write-Host "Successfully extracted Lego to: $legoDestination" -ForegroundColor Green

# Step 5: Add to PATH if needed
$pathScope = "User"  # Default to User scope

# If running as admin, ask if they want to add to Machine PATH instead
if (Test-Admin) {
    $addToMachinePath = Read-Host "Would you like to add Lego to the Machine PATH instead of User PATH? (y/N)"
    if ($addToMachinePath -eq "y" -or $addToMachinePath -eq "Y") {
        $pathScope = "Machine"
    }
}

Add-ToPath -PathToAdd $boxDirectory -Scope $pathScope

# Step 6: Verify installation
Write-Host "Verifying Lego installation..." -ForegroundColor Cyan
try {
    $legoVersion = & $legoDestination --version
    Write-Host "Lego installed successfully!" -ForegroundColor Green
    Write-Host "Version: $legoVersion" -ForegroundColor Green
} catch {
    Write-Host "Lego installed but version check failed. You may need to restart your terminal." -ForegroundColor Yellow
}

# Step 7: Add troubleshooting information for common DNS providers
Write-Host ""
Write-Host "===============================================================" -ForegroundColor Cyan
Write-Host "COMMON TROUBLESHOOTING TIPS" -ForegroundColor Yellow
Write-Host "---------------------------------------------------------------" -ForegroundColor Yellow

Write-Host "Cloudflare API Issues:" -ForegroundColor Yellow
Write-Host "1. For 'Invalid request headers' errors:" -ForegroundColor White
Write-Host "   - Verify your Cloudflare API token has 'Zone:Read' and 'DNS:Edit' permissions" -ForegroundColor White
Write-Host "   - Ensure token is created for the specific zone or 'All zones'" -ForegroundColor White
Write-Host "   - Check that you're using API Token (recommended) not Global API Key" -ForegroundColor White
Write-Host "   - Example ENV file setup:" -ForegroundColor White
Write-Host "     CLOUDFLARE_DNS_API_TOKEN=your_api_token_here" -ForegroundColor White
Write-Host "     DNS_PROVIDER=cloudflare" -ForegroundColor White
Write-Host "     DOMAINS=example.com,*.example.com" -ForegroundColor White
Write-Host "     LEGO_EMAIL=your_email@example.com" -ForegroundColor White
Write-Host ""
Write-Host "2. To verify Cloudflare API token works:" -ForegroundColor White
Write-Host "   - Run this PowerShell command:" -ForegroundColor White
Write-Host "     Invoke-RestMethod -Uri 'https://api.cloudflare.com/client/v4/user/tokens/verify' -Headers @{'Authorization'='Bearer YOUR_API_TOKEN'}" -ForegroundColor White
Write-Host "   - You should see a success response with status=true" -ForegroundColor White

Write-Host "---------------------------------------------------------------" -ForegroundColor Yellow
Write-Host "EasyDNS API Issues:" -ForegroundColor Yellow
Write-Host "1. Required environment variables:" -ForegroundColor White
Write-Host "   EASYDNS_TOKEN=your_token" -ForegroundColor White
Write-Host "   EASYDNS_KEY=your_key" -ForegroundColor White
Write-Host "   DNS_PROVIDER=easydns" -ForegroundColor White
Write-Host "   DOMAINS=example.com,*.example.com" -ForegroundColor White
Write-Host "   LEGO_EMAIL=your_email@example.com" -ForegroundColor White

Write-Host "===============================================================" -ForegroundColor Cyan

Write-Host ""
Write-Host "===============================================================" -ForegroundColor Cyan
Write-Host "Lego ACME client has been installed alongside CommandBox." -ForegroundColor Cyan
Write-Host "Location: $legoDestination" -ForegroundColor Cyan
Write-Host "Added to $pathScope PATH: $boxDirectory" -ForegroundColor Cyan
Write-Host ""
Write-Host "You may need to restart your terminal for PATH changes to take effect." -ForegroundColor Yellow
Write-Host "Test installation with: lego --version" -ForegroundColor Yellow
Write-Host "===============================================================" -ForegroundColor Cyan
```

## Running the Script

1. Save the script as `Install-Lego.ps1`
2. Open PowerShell as administrator (recommended)
3. Navigate to the directory containing the script
4. Run the script:
   ```
   .\Install-Lego.ps1
   ```
5. Follow any on-screen prompts

## How It Works

The script performs the following actions:

1. **Fetches the latest Lego version** from GitHub using the API
2. **Locates your CommandBox installation** by searching common installation paths
3. **Downloads the appropriate Lego binary** for Windows (64-bit)
4. **Extracts the executable** to your CommandBox directory
5. **Adds the directory to your PATH** (user or machine-level, you choose)
6. **Verifies the installation** by running a version check

If any step fails, the script provides detailed error information.

## Environment Variables for DNS Providers

Lego requires specific environment variables to be set depending on your DNS provider. The script includes common examples for:

### Cloudflare

```
CLOUDFLARE_DNS_API_TOKEN=your_api_token_here
DNS_PROVIDER=cloudflare
DOMAINS=example.com,*.example.com
LEGO_EMAIL=your_email@example.com
```

### EasyDNS

```
EASYDNS_TOKEN=your_token
EASYDNS_KEY=your_key
DNS_PROVIDER=easydns
DOMAINS=example.com,*.example.com
LEGO_EMAIL=your_email@example.com
```

## Troubleshooting

- **CommandBox Not Found**: Ensure CommandBox is installed before running the script
- **PATH Updates Not Working**: Restart your terminal or PowerShell session
- **Permission Issues**: Run PowerShell as Administrator
- **API Errors with DNS Providers**: Verify your API tokens have correct permissions

## Further Resources

- [Lego GitHub Repository](https://github.com/go-acme/lego)
- [Lego Documentation](https://go-acme.github.io/lego/)
- [CommandBox Documentation](https://commandbox.ortusbooks.com/)
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)