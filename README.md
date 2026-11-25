# Windows-Autounattend-Re-imaging-Process
Windows 11 Reimaging process with rufus / XML / PowerShell
# Windows 11 Enterprise Zero-Driver USB Deployment (Mixed Lenovo & Surface Fleet)

Standardized bare-metal USB deployment for mixed enterprise fleets running **Lenovo ThinkPad (X1 / X13)** and **Microsoft Surface Laptop (5, 6 for Business, and 7 Intel)** endpoints.

## Problem Solved

Offline DISM driver injection and `autounattend.xml` PnP driver paths frequently fail to bind Surface input hardware during the Out-of-Box Experience (OOBE). Surface built-in keyboards and touchpads route through the Intel Serial IO (I2C/SPI) bus, Surface Serial Hub, and Surface System Aggregator Module (SAM).

This workflow avoids bloating deployment USBs with 4–6 GB of extracted Surface driver packages by leveraging `SetupComplete.cmd` (executed automatically under Volume License / Enterprise GVLK keys immediately prior to OOBE) to detect Surface hardware and install the official Windows 11 Surface MSI online.

## Supported Hardware

| Hardware Family | Behavior During Deployment |
| :--- | :--- |
| **Surface Laptop 5** | Detects SKU/model, downloads Windows 11 MSI (`id=104679`), installs silently |
| **Surface Laptop 6 for Business** | Detects SKU/model, downloads Windows 11 MSI (`id=105946`), installs silently |
| **Surface Laptop 7 for Business (Intel)** | Detects AMD64 SKU/model, downloads Windows 11 MSI (`id=108014`), installs silently |
| **Lenovo ThinkPad X1 / X13 (All Gens)** | Skips Surface script immediately; proceeds directly to OOBE / Autopilot |
| **ARM64 / Snapdragon Devices** | Skipped (`PROCESSOR_ARCHITECTURE -ne 'AMD64'`) |

## Prerequisites

* **USB Flash Drive:** 32 GB or larger.
* **Rufus 3.17 or newer (4.x recommended):** Required for the Microsoft-signed `UEFI:NTFS` bootloader so Secure Boot remains enabled on Surface and Lenovo hardware.
* **Windows 11 Enterprise ISO:** Clean, unmodified `boot.wim` and `install.wim` (no offline-injected drivers).
* **Wired Network:** USB-C Ethernet adapter using an inbox Windows 11 driver with internet access to `www.microsoft.com` and `download.microsoft.com` during imaging.

## USB Directory Structure

```text
D:\
├── Autounattend.xml
└── sources\
    ├── boot.wim
    ├── install.wim
    └── $OEM$\ └── $$\
            └── Setup\
                └── Scripts\
                    ├── SetupComplete.cmd
                    └── Install-SurfaceOnline.ps1
```

Windows Setup automatically copies `sources\$OEM$\$$\Setup\Scripts\` to `C:\Windows\Setup\Scripts\` on the target OS volume.

## Deployment Files

### 1. `Autounattend.xml`

Place at the root of the USB (`D:\Autounattend.xml`). Wipes Disk 0, configures GPT partitions, and installs **Windows 11 Enterprise** using Microsoft's public KMS Client Setup Key (GVLK) to guarantee `SetupComplete.cmd` execution.

> **Partition Note:** The configuration below uses a 100 MB EFI partition matching the baseline template. On Windows 11 25H2+ and Surface devices that stage UEFI firmware capsules on the EFI partition, increasing `<Size>100</Size>` to `<Size>260</Size>` is recommended. Adding a dedicated Recovery (`WinRE`) partition is also required if Intune **Autopilot Reset** will be used later (`reagentc /info` must report `Enabled`).

```xml
<?xml version="1.0" encoding="utf-8"?>
<unattend xmlns="urn:schemas-microsoft-com:unattend" xmlns:wcm="[http://schemas.microsoft.com/WMIConfig/2002/State](http://schemas.microsoft.com/WMIConfig/2002/State)">
    <settings pass="windowsPE">
        <component name="Microsoft-Windows-International-Core-WinPE" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
            <SetupUILanguage>
                <UILanguage>en-US</UILanguage>
            </SetupUILanguage>
            <InputLocale>0409:00000409</InputLocale>
            <SystemLocale>en-US</SystemLocale>
            <UILanguage>en-US</UILanguage>
            <UserLocale>en-US</UserLocale>
        </component>
        <component name="Microsoft-Windows-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
            <DiskConfiguration>
                <Disk wcm:action="add">
                    <DiskID>0</DiskID>
                    <WillWipeDisk>true</WillWipeDisk>
                    <CreatePartitions>
                        <CreatePartition wcm:action="add">
                            <Order>1</Order>
                            <Type>EFI</Type>
                            <Size>100</Size>
                        </CreatePartition>
                        <CreatePartition wcm:action="add">
                            <Order>2</Order>
                            <Type>MSR</Type>
                            <Size>16</Size>
                        </CreatePartition>
                        <CreatePartition wcm:action="add">
                            <Order>3</Order>
                            <Type>Primary</Type>
                            <Extend>true</Extend>
                        </CreatePartition>
                    </CreatePartitions>
                    <ModifyPartitions>
                        <ModifyPartition wcm:action="add">
                            <Order>1</Order>
                            <PartitionID>1</PartitionID>
                            <Format>FAT32</Format>
                        </ModifyPartition>
                        <ModifyPartition wcm:action="add">
                            <Order>2</Order>
                            <PartitionID>2</PartitionID>
                        </ModifyPartition>
                        <ModifyPartition wcm:action="add">
                            <Order>3</Order>
                            <PartitionID>3</PartitionID>
                            <Format>NTFS</Format>
                        </ModifyPartition>
                    </ModifyPartitions>
                </Disk>
            </DiskConfiguration>
            <ImageInstall>
                <OSImage>
                    <InstallTo>
                        <DiskID>0</DiskID>
                        <PartitionID>3</PartitionID>
                    </InstallTo>
                    <InstallFrom>
                        <MetaData wcm:action="add">
                            <Key>/IMAGE/NAME</Key>
                            <Value>Windows 11 Enterprise</Value>
                        </MetaData>
                    </InstallFrom>
                </OSImage>
            </ImageInstall>
            <UserData>
                <AcceptEula>true</AcceptEula>
                <ProductKey>
                    <Key>NPPR9-FWDCX-D2C8J-H872K-2YT43</Key>
                </ProductKey>
            </UserData>
        </component>
    </settings>
</unattend>
```

### 2. `SetupComplete.cmd`

Save to `D:\sources\$OEM$\$$\Setup\Scripts\SetupComplete.cmd`:

```cmd
@echo off
powershell.exe -NoProfile -ExecutionPolicy Bypass -File "%WINDIR%\Setup\Scripts\Install-SurfaceOnline.ps1"
```

### 3. `Install-SurfaceOnline.ps1`

Save to `D:\sources\$OEM$\$$\Setup\Scripts\Install-SurfaceOnline.ps1`:

```powershell
$ErrorActionPreference = 'Stop'$logDir  = 'C:\ProgramData\Reimage\SurfaceOnline'
New-Item -ItemType Directory -Force -Path $logDir | Out-Null
$logFile = Join-Path$logDir 'Install.log'

function Write-Log([string]$Msg) {
    Add-Content -Path $logFile -Value ("[{0}] {1}" -f (Get-Date -Format 'yyyy-MM-dd HH:mm:ss'), $Msg)
}

try {
    $cs    = Get-CimInstance -ClassName Win32_ComputerSystem$mfg   = "$($cs.Manufacturer)"
    $sku   = "$($cs.SystemSKUNumber)"
    $model = "$($cs.Model)"
    $arch  = "$env:PROCESSOR_ARCHITECTURE"

    Write-Log "Manufacturer=$mfg \vert{} Model=$model | SKU=$sku \vert{} Arch=$arch"

    if ($mfg -notmatch 'Microsoft' -or$arch -ne 'AMD64') {
        Write-Log 'Non-Surface or non-AMD64 hardware. Exiting.'
        exit 0
    }

    $target =$null
    if ($sku -like 'Surface_Laptop_5*' -or $model -match 'Surface Laptop 5') {$target = @{ Name = 'Surface5'; Url = '[https://www.microsoft.com/en-us/download/details.aspx?id=104679](https://www.microsoft.com/en-us/download/details.aspx?id=104679)'; Pattern = 'href="(https://download\.microsoft\.com/download/[^"]+/SurfaceLaptop5_Win11_[^"]+\.msi)"' }
    } elseif ($sku -like 'Surface_Laptop_6*' -or $model -match 'Surface Laptop 6') {$target = @{ Name = 'Surface6'; Url = '[https://www.microsoft.com/en-us/download/details.aspx?id=105946](https://www.microsoft.com/en-us/download/details.aspx?id=105946)'; Pattern = 'href="(https://download\.microsoft\.com/download/[^"]+/SurfaceLaptop6forBusiness_Win11_[^"]+\.msi)"' }
    } elseif ($sku -like 'Surface_Laptop_7*' -or $model -match 'Surface Laptop 7') {$target = @{ Name = 'Surface7'; Url = '[https://www.microsoft.com/en-us/download/details.aspx?id=108014](https://www.microsoft.com/en-us/download/details.aspx?id=108014)'; Pattern = 'href="(https://download\.microsoft\.com/download/[^"]+/SurfaceLaptop7withIntel_Win11_[^"]+\.msi)"' }
    } else {
        Write-Log "Unmapped Surface model/SKU. Exiting."
        exit 0
    }

    # Wait up to 60 seconds for wired DNS/network initialization
    $online =$false
    for ($i = 0; $i -lt 20; $i++) {
        try {
            [void][System.Net.Dns]::GetHostAddresses('download.microsoft.com')
            $online =$true
            break
        } catch {
            Start-Sleep -Seconds 3
        }
    }
    if (-not $online) { throw 'No DNS/internet connectivity within 60 seconds.' }

    [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
    Write-Log "Resolving Win11 MSI from $($target.Url)"
    $html = (Invoke-WebRequest -Uri$target.Url -UseBasicParsing).Content
    if ($html -notmatch$target.Pattern) { throw "Win11 MSI link extraction failed for $($target.Name)." }

    $msiUrl  = $matches[1]$msiPath = Join-Path 'C:\Windows\Temp' "$($target.Name)_Win11.msi"
    $msiLog  = Join-Path$logDir "$($target.Name)_msiexec.log"

    Write-Log "Downloading $msiUrl"
    (New-Object System.Net.WebClient).DownloadFile($msiUrl,$msiPath)

    Write-Log "Installing $msiPath"
    $proc = Start-Process -FilePath 'msiexec.exe' -ArgumentList "/i `"$msiPath`" /qn /norestart /L*v `"$msiLog`"" -Wait -PassThru
    Write-Log "msiexec exit code: $($proc.ExitCode)"

    if ($proc.ExitCode -notin 0, 3010) {
        throw "MSI installation failed with exit code $($proc.ExitCode)."
    }
}
catch {
    Write-Log "ERROR: $($_.Exception.Message)"
}
finally {
    if ($msiPath -and (Test-Path$msiPath)) {
        Remove-Item -Path $msiPath -Force -ErrorAction SilentlyContinue
    }
}
```

## Creating & Cloning Deployment USBs

### Initialize a Blank USB (Rufus 4.x)

1. Insert a 32 GB+ USB flash drive.
2. Open **Rufus 4.x** and select the drive under **Device**.
3. Expand **Show advanced drive properties**.
4. Configure:
   * **Boot selection:** `UEFI:NTFS`
   * **Partition scheme:** `GPT`
   * **Target system:** `UEFI (non CSM)`
   * **File system:** `NTFS`
   * **Cluster size:** `4096 bytes (Default)`
5. Set a volume label (e.g., `BootMedia01`) and click **START**.

### Clone Master USB to Target USB

After Rufus finishes, mirror the verified master USB (`D:\`) to the newly initialized USB (`F:\`) in an elevated PowerShell prompt. Double-check drive letters before running—reversing them overwrites the master drive.

```powershell
robocopy D:\ F:\ /MIR /R:3 /W:1
```

## Deployment Execution & Exit Code `3010`

1. Connect wired Ethernet and the deployment USB to the endpoint and boot to USB.
2. Windows Setup wipes Disk 0, installs Windows 11 Enterprise, and executes `SetupComplete.cmd` before OOBE (adds 3–5 minutes on Surface endpoints for MSI download and installation).
3. Surface driver MSIs typically return exit code `3010` (`ERROR_SUCCESS_REBOOT_REQUIRED`) because Intel Serial IO and SAM bus drivers require one kernel restart to bind.
4. When the Surface reaches the OOBE country/region screen, **power-cycle the device once** using the physical power button. Built-in keyboard and touchpad input will initialize immediately upon returning to OOBE—no external peripherals required.

## Logs & Verification

Press `Shift + F10` at OOBE and inspect the deployment log:

```powershell
Get-Content 'C:\ProgramData\Reimage\SurfaceOnline\Install.log'
```

* **Success:** `msiexec exit code: 0` or `msiexec exit code: 3010`
* **Detailed MSI Log:** `C:\ProgramData\Reimage\SurfaceOnline\Surface*_msiexec.log`
* **Syntax Check (Technician PC):**

```powershell
[xml](Get-Content 'D:\Autounattend.xml') | Out-Null
[System.Management.Automation.Language.Parser]::ParseFile('D:\sources\$OEM$\$$\Setup\Scripts\Install-SurfaceOnline.ps1', [ref]$null, [ref]$errors);$errors
```
