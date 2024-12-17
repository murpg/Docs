# Creating Additional ColdFusion Instances in a security baseline Environment

This guide explains how to create additional ColdFusion instances when your environment is STIG-compliant and using custom service accounts instead of the default System account.

## Registry Permissions Setup

When ColdFusion services are running under a custom user account instead of System, you need to grant the service account permissions to create new services. This requires modifying registry permissions in three locations:

1. `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services`
2. `HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Services`
3. `HKEY_LOCAL_MACHINE\SYSTEM\ControlSet002\Services`

For each path:
1. Open Registry Editor (regedit)
2. Navigate to the Services key
3. Right-click and select Permissions
4. Click Add
5. Select the custom ColdFusion service account
6. Grant Full Control permissions

After modifying permissions, restart all ColdFusion services.

## Apply permissions with Powershell

## Requires running as Administrator

```
function Set-RegistryPermissions {
    param (
        [Parameter(Mandatory=$true)]
        [string]$ServiceAccount,
        
        [Parameter(Mandatory=$false)]
        [string[]]$RegistryPaths = @(
            'HKLM:\SYSTEM\CurrentControlSet\Services',
            'HKLM:\SYSTEM\ControlSet001\Services',
            'HKLM:\SYSTEM\ControlSet002\Services'
        )
    )
    
    try {
        # Load the required type
        $identity = New-Object System.Security.Principal.NTAccount($ServiceAccount)
        
        foreach ($registryPath in $RegistryPaths) {
            Write-Host "Processing $registryPath..."
            
            # Check if path exists
            if (-not (Test-Path $registryPath)) {
                Write-Warning "Registry path does not exist: $registryPath"
                continue
            }
            
            # Get existing ACL
            $acl = Get-Acl $registryPath
            
            # Create new rule
            $rights = [System.Security.AccessControl.RegistryRights]::FullControl
            $inheritance = [System.Security.AccessControl.InheritanceFlags]"ContainerInherit,ObjectInherit"
            $propagation = [System.Security.AccessControl.PropagationFlags]::None
            $type = [System.Security.AccessControl.AccessControlType]::Allow
            
            try {
                $rule = New-Object System.Security.AccessControl.RegistryAccessRule($identity, $rights, $inheritance, $propagation, $type)
            }
            catch {
                throw "Failed to create access rule for account '$ServiceAccount'. Please verify the account exists. Error: $_"
            }
            
            # Add new rule to ACL
            $acl.AddAccessRule($rule)
            
            # Apply the new ACL
            try {
                $acl | Set-Acl -Path $registryPath
                Write-Host "Successfully applied permissions for $ServiceAccount to $registryPath" -ForegroundColor Green
            }
            catch {
                throw "Failed to apply permissions to $registryPath. Error: $_"
            }
        }
        
        Write-Host "`nPermissions have been applied successfully to all registry paths." -ForegroundColor Green
        Write-Host "Please restart all ColdFusion services for the changes to take effect." -ForegroundColor Yellow
        
        # Optionally restart ColdFusion services
        $restartServices = Read-Host "Would you like to restart ColdFusion services now? (y/n)"
        if ($restartServices -eq 'y') {
            Write-Host "Restarting ColdFusion services..."
            Get-Service "ColdFusion 2021*" | Stop-Service -Force
            Start-Sleep -Seconds 5
            Get-Service "ColdFusion 2021*" | Start-Service
            Write-Host "Services have been restarted." -ForegroundColor Green
        }
    }
    catch {
        Write-Error "An error occurred: $_"
        return $false
    }
}

```

## Example usage:
## Set-RegistryPermissions -ServiceAccount "DOMAIN\CFServiceAccount"
## Or for a local account:
## Set-RegistryPermissions -ServiceAccount ".\CFServiceAccount"

## Creating the New Instance

1. Create the new instance through the ColdFusion Administrator
2. Use the following PowerShell script to properly set up the service (replace "cfusiontwo" with your instance name)  
3. If your ColdFusion instances are not on D:\ColdFusion2021 you will need to update the script below

```powershell
# Function to force stop a service and its processes
function Force-StopService {
    param (
        [string]$serviceName
    )
    
    Write-Host "Forcing stop of service: $serviceName"
    
    # Try normal service stop first
    $service = Get-Service $serviceName -ErrorAction SilentlyContinue
    if ($service) {
        Stop-Service $serviceName -Force
        $service.WaitForStatus('Stopped', '00:00:30')
        
        # If still not stopped, get aggressive
        if ((Get-Service $serviceName).Status -ne 'Stopped') {
            Write-Host "Service didn't stop normally, using forceful methods..."
            
            # Get process ID from service
            $processId = (Get-WmiObject Win32_Service -Filter "Name='$($service.Name)'").ProcessId
            
            if ($processId -gt 0) {
                # Kill the process
                Write-Host "Killing process ID: $processId"
                taskkill /PID $processId /F
            }
            
            # Use sc.exe to force stop
            Write-Host "Using sc.exe to force stop..."
            & sc.exe stop $serviceName
            Start-Sleep -Seconds 2
            
            # Final check
            if ((Get-Service $serviceName).Status -ne 'Stopped') {
                throw "Failed to stop service $serviceName"
            }
        }
    }
}

# Stop all CF related services
Write-Host "Stopping all ColdFusion services..."
$cfServices = Get-Service "ColdFusion 2021*"
foreach ($service in $cfServices) {
    Force-StopService $service.Name
}

# Double check for any running CF processes
Get-Process | Where-Object {$_.Name -like "*coldfusion*" -or $_.Name -like "cf*"} | ForEach-Object {
    Write-Host "Killing process: $($_.Name) (ID: $($_.Id))"
    Stop-Process -Id $_.Id -Force
}

# Wait to ensure everything is stopped
Start-Sleep -Seconds 5

# Verify all services are stopped
$runningServices = Get-Service "ColdFusion 2021*" | Where-Object {$_.Status -ne 'Stopped'}
if ($runningServices) {
    throw "Failed to stop all ColdFusion services. Still running: $($runningServices.Name -join ', ')"
}

Write-Host "All ColdFusion services confirmed stopped."

# Configuration
$sourcePath = "D:\ColdFusion2021\cfusion"
$destPath = "D:\ColdFusion2021\cfusiontwo"
$serviceName = "ColdFusion 2021 Application Server cfusiontwo"
$displayName = "ColdFusion 2021 Application Server cfusiontwo"
$description = "ColdFusion 2021 cfusiontwo instance service"
$binaryPath = "D:\ColdFusion2021\cfusiontwo\bin\coldfusionsvc.exe"

# Create directories
Write-Host "Creating directories..."
@("bin", "lib", "runtime\lib") | ForEach-Object {
    $dir = Join-Path $destPath $_
    if (-not (Test-Path $dir)) {
        New-Item -ItemType Directory -Path $dir -Force
        Write-Host "Created directory: $_"
    }
}

# Copy essential files
Write-Host "Copying essential files..."
@(
    "bin\cf-bootstrap.jar",
    "bin\cf-startup.jar",
    "bin\coldfusionsvc.exe",
    "bin\jvm.config"
) | ForEach-Object {
    $source = Join-Path $sourcePath $_
    $dest = Join-Path $destPath $_
    if (Test-Path $source) {
        Copy-Item $source $dest -Force
        Write-Host "Copied: $_"
    }
}

# Copy lib directories with verification
Write-Host "Copying lib files..."
$maxRetries = 3
$retryCount = 0
$success = $false

while (-not $success -and $retryCount -lt $maxRetries) {
    try {
        Copy-Item "$sourcePath\lib\*" "$destPath\lib\" -Recurse -Force -ErrorAction Stop
        $success = $true
    }
    catch {
        $retryCount++
        Write-Host "Failed to copy lib files (Attempt $retryCount of $maxRetries): $($_.Exception.Message)"
        Start-Sleep -Seconds 5
        
        # On retry, check for and kill any processes locking files
        Write-Host "Checking for processes locking files..."
        Get-Process | Where-Object {$_.Name -like "*coldfusion*" -or $_.Name -like "cf*"} | 
        Stop-Process -Force
    }
}

if (-not $success) {
    throw "Failed to copy lib files after $maxRetries attempts"
}

Write-Host "Copying runtime\lib files..."
Copy-Item "$sourcePath\runtime\lib\*" "$destPath\runtime\lib\" -Recurse -Force

Write-Host "File copy complete. Creating service..."

# Create the Windows service
if (Get-Service $serviceName -ErrorAction SilentlyContinue) {
    Force-StopService $serviceName
    Write-Host "Removing existing service..."
    sc.exe delete $serviceName
    Start-Sleep -Seconds 2
}

# Create service with dependencies
$result = sc.exe create "$serviceName" binPath= "$binaryPath" start= auto DisplayName= "$displayName" depend= "Tcpip/Afd"
if ($result -match "SUCCESS") {
    # Configure service
    sc.exe description "$serviceName" "$description"
    sc.exe config "$serviceName" obj= "LocalSystem"
    sc.exe failure "$serviceName" reset= 86400 actions= restart/60000/restart/60000/restart/60000
    
    Write-Host "Service created successfully"
    
    # Start services
    Write-Host "Starting services..."
    Start-Service "ColdFusion 2021 Application Server"
    Start-Sleep -Seconds 10
    Start-Service $serviceName
    
    Get-Service "ColdFusion 2021*" | Format-Table Name, DisplayName, Status, StartType
} else {
    throw "Failed to create service: $result"
}
```

## Service Account Configuration

If the new instance is running under the System account, update it to use your custom service account:

```powershell
sc.exe config "ColdFusion 2021 Application Server instancename" obj= "DOMAIN\username" password= "password"
```

## Verifying Services

Check the status of all ColdFusion services:

```powershell
Get-Service "ColdFusion 2021*" | Format-Table Name, DisplayName, Status, StartType
```

Expected output should show all services running:
```
Name                                       DisplayName                                 Status StartType
----                                       -----------                                 ------ ---------
ColdFusion 2021 .NET Service               ColdFusion 2021 .NET Service               Running Automatic
ColdFusion 2021 Application Server         ColdFusion 2021 Application Server         Running Automatic
ColdFusion 2021 Application Server cfusiontwo ColdFusion 2021 Application Server cfusiontwo Running Automatic
ColdFusion 2021 ODBC Agent                 ColdFusion 2021 ODBC Agent                 Running Automatic
ColdFusion 2021 ODBC Server                ColdFusion 2021 ODBC Server                Running Automatic
```

## Service Management Script

For CI/CD purposes, you can use this service management script to control all ColdFusion services:

```powershell
# Function to nicely format and display CF services status
function Show-CFServices {
    Write-Host "`nColdFusion Services Status:"
    Write-Host "============================="
    Get-Service "ColdFusion 2021*" | Format-Table Name, DisplayName, Status, StartType
    Write-Host ""
}

# Function to stop all CF services with verification
function Stop-AllCFServices {
    Write-Host "Stopping all ColdFusion services..."
    $services = Get-Service "ColdFusion 2021*"
    
    foreach ($service in $services) {
        Write-Host "Stopping $($service.DisplayName)..."
        if ($service.Status -eq 'Running') {
            Stop-Service $service.Name -Force
            $maxWait = 30
            $waited = 0
            while (((Get-Service $service.Name).Status -ne 'Stopped') -and ($waited -lt $maxWait)) {
                Write-Host "Waiting for service to stop..."
                Start-Sleep -Seconds 1
                $waited++
            }
            
            # If service still running, try more aggressive methods
            if ((Get-Service $service.Name).Status -ne 'Stopped') {
                Write-Host "Service didn't stop gracefully, forcing stop..."
                $processId = (Get-WmiObject Win32_Service -Filter "Name='$($service.Name)'").ProcessId
                if ($processId) {
                    taskkill /PID $processId /F
                }
                & sc.exe stop $service.Name
            }
        }
    }
    
    # Final verification
    $stillRunning = Get-Service "ColdFusion 2021*" | Where-Object {$_.Status -eq 'Running'}
    if ($stillRunning) {
        Write-Host "WARNING: Some services are still running:"
        $stillRunning | Format-Table Name, DisplayName, Status
        return $false
    } else {
        Write-Host "All ColdFusion services stopped successfully."
        return $true
    }
}

# Function to start all CF services with verification
function Start-AllCFServices {
    Write-Host "Starting all ColdFusion services..."
    
    # Define service start order
    $serviceOrder = @(
        "ColdFusion 2021 ODBC Server",
        "ColdFusion 2021 ODBC Agent",
        "ColdFusion 2021 Application Server",
        "ColdFusion 2021 Application Server cfusiontwo",
        "ColdFusion 2021 .NET Service"
    )
    
    foreach ($serviceName in $serviceOrder) {
        $service = Get-Service $serviceName -ErrorAction SilentlyContinue
        if ($service) {
            Write-Host "Starting $serviceName..."
            Start-Service $serviceName
            Start-Sleep -Seconds 2
        }
    }
    
    # Verify all services started
    $notRunning = Get-Service "ColdFusion 2021*" | Where-Object {$_.Status -ne 'Running'}
    if ($notRunning) {
        Write-Host "WARNING: Some services failed to start:"
        $notRunning | Format-Table Name, DisplayName, Status
        return $false
    } else {
        Write-Host "All ColdFusion services started successfully."
        return $true
    }
}

# Show initial status
Show-CFServices

# Stop all services
$stopResult = Stop-AllCFServices
if (-not $stopResult) {
    Write-Host "Failed to stop all services. Aborting."
    exit 1
}

Show-CFServices

# Optional: Uncomment to also start services
# $startResult = Start-AllCFServices
# Show-CFServices
```

This script provides functions for:
- Displaying service status
- Stopping all ColdFusion services with verification
- Starting services in the correct order
- Verifying service states

Save this as `cf-service-manager.ps1` and run it as administrator when needed.

## Common Issues

1. **Service Creation Fails**: Verify registry permissions for the service account
2. **Service Won't Start**: Check Windows Event Viewer for specific errors
3. **Files Locked**: Use the service management script to properly stop all services
4. **Service Account Issues**: Ensure the service account has proper permissions on the instance directories

## Best Practices

1. Always use the service management script to stop/start services
2. Verify service account permissions before creating new instances
3. Follow the correct service start order (ODBC Server → ODBC Agent → Application Server → Additional Instances → .NET Service)
4. Maintain consistent service account usage across all ColdFusion services