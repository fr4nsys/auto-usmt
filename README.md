# auto-usmt
 PowerShell script to migrate the complete profile of a local user to the domain user on the computer with USMT (User State Migration Tool), along with the domain check and the option to add the computer to the domain. The script includes disk space verification, certificate export and user profile migration with USMT.
 
This PowerShell script performs user profile migration using the User State Migration Tool (USMT). It includes functions to get the size of a folder and export certificates. The script checks if the computer is already in a specific domain (represented by the variables $exampleDomain and $exampleOrganization). If it's in the domain, it migrates the user profile. If not, it provides an option to add the computer to the domain.

```powershell
# Function to get the size of a folder
function Get-FolderSize {
    param ([string]$Path)
    (Get-ChildItem -Path $Path -Recurse -Force -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1GB
}

# Function to Export Certificates
function Export-Certificates {
    param (
        [string]$ExportPath,
        [System.Security.SecureString]$ExportPassword
    )

    $exportType = [System.Security.Cryptography.X509Certificates.X509ContentType]::Pkcs12
    $store = New-Object System.Security.Cryptography.X509Certificates.X509Store "My", "CurrentUser"
    $store.Open([System.Security.Cryptography.X509Certificates.OpenFlags]::ReadOnly)

    foreach ($cert in $store.Certificates) {
        if ($cert.HasPrivateKey) {
            $exportPathFull = Join-Path $ExportPath ($cert.Thumbprint + ".pfx")
            $exportedBytes = $cert.Export($exportType, $ExportPassword)
            [System.IO.File]::WriteAllBytes($exportPathFull, $exportedBytes)
            Write-Host "Certificate exported: $exportPathFull"
        }
    }

    $store.Close()
}

# USMT Configuration
$USMTPath = "C:\Users\Public\profile_USMT\domain_usmt_bin"
$USMTStorePath = "C:\Users\Public\profile_USMT"

# Get computer information and check domain
$hostname = Get-ComputerInfo | Select-Object -ExpandProperty CsName
$checkdomain = (Get-WmiObject -Class Win32_ComputerSystem).Domain

# Example Variables
$exampleDomain = "domain.dom"
$exampleOrganization = "DOMAIN"

# Domain check and migration process
if ($checkdomain -match $exampleDomain) {
    Write-Host "The computer is already in the $exampleOrganization domain."

    # Request user information for migration
    $actualuser = Read-Host -Prompt "Enter the username (current, local) you want to migrate"
    $newuser = Read-Host -Prompt "Enter the username (new, domain) where the profile will be migrated"

    # Check disk space before migration
    $UserProfilePath = "C:\Users\$actualuser"
    $UserProfileSizeGB = Get-FolderSize -Path $UserProfilePath
    $RequiredSpaceGB = $UserProfileSizeGB * 2 # Estimate required space
    $FreeSpaceGB = (Get-PSDrive C).Free / 1GB

    if ($FreeSpaceGB -lt $RequiredSpaceGB) {
        Write-Host "Not enough disk space to continue. Free space: $FreeSpaceGB GB. Required space: $RequiredSpaceGB GB."
        exit
    }

    # Export Certificates
    $ExportPassword = Read-Host "Enter a secure password to export the certificates" -AsSecureString
    Export-Certificates -ExportPath $USMTStorePath -ExportPassword $ExportPassword

    # Run ScanState
    $ScanStateCommand = "$USMTPath\ScanState.exe $USMTStorePath /ue:`"$hostname\*`" /ue:`"$exampleOrganization\*`" /ui:`"$hostname\$actualuser`" /i:$USMTPath\MigDocs.xml /i:$USMTPath\MigApp.xml /i:$USMTPath\MigUser.xml /i:$USMTPath\only_c.xml /v:13 /l:$USMTPath\ScanState.log /localonly /efs:copyraw /o"
    Invoke-Expression $ScanStateCommand

    # Run LoadState
    $LoadStateCommand = "$USMTPath\LoadState.exe $USMTStorePath /mu:`"$hostname\$actualuser`":`"$exampleOrganization\$newuser`" /i:$USMTPath\MigDocs.xml /i:$USMTPath\MigApp.xml /i:$USMTPath\MigUser.xml /v:13 /l:$USMTPath\LoadState.log /c"
    Invoke-Expression $LoadStateCommand

    Write-Host "USMT migration process completed."
} else {
    $askadddomain = Read-Host -Prompt "The computer is not in the $exampleOrganization domain. Type 'y' to add it to the domain. Press 'n' to cancel the process"

    if ($askadddomain -match 'y') {
        $addomain = $exampleDomain
        $adUserName = "domain\administrator"
        $adPassword = "Password" | ConvertTo-SecureString -AsPlainText -Force
        $adCredential = New-Object System.Management.Automation.PSCredential -ArgumentList $adUserName, $adPassword
        Add-Computer -DomainName $addomain -DomainCredential $adCredential -Restart -Verbose
        Write-Host "The computer will be added to the $exampleOrganization domain and will restart."
    }

    if ($askadddomain -match 'n') {
        Write-Host "Process canceled."
        exit
    }
}
```
