---
layout: post
title: Hanlon now supports Windows provisioning!
---


# WinPE Build Process

We tried to make this as painless as possible.  




{% highlight PowerShell linenos %}

#!powershell

###
#
# Utilized Code various code from multiple locations
# Sources:
# http://stackoverflow.com/questions/5648931/test-if-registry-value-exists
# https://github.com/puppetlabs/razor-server/blob/master/build-winpe/build-razor-winpe.ps1
###


###
# Define Local Static Variables
# winpeCabs - These are all the CABS that need to be installed to install Windows
###


$DebugPreference = "Continue"

$PackageCabs = @( "WinPE-WMI.cab", "WinPE-NetFx.cab",
				"WinPE-Scripting.cab", "WinPE-PowerShell.cab",
				"WinPE-Setup.cab", "WinPE-Setup-Server.cab")

$LangPackageCabs = @( "lp.cab",
                      "WinPE-Setup_en-us.cab",
                      "WinPE-Setup-Server_en-us.cab" )

$SubPathWinPeImage = "Assessment and Deployment Kit\Windows Preinstallation Environment\amd64\en-us\winpe.wim"
$SubPathPackages = "Assessment and Deployment Kit\Windows Preinstallation Environment\amd64\WinPE_OCs"
$SubPathLangPackages = "Assessment and Deployment Kit\Windows Preinstallation Environment\amd64\WinPE_OCs\en-us"

$MountPath = "$env:SystemDrive\mount-point"
$WimPath = "$env:SystemDrive\winpe"
$ScriptPath = "$env:SystemDrive\script"
$DriversPath = "$env:SystemDrive\drivers"

$paths = @($MountPath,$WimPath,$ScriptPath,$DriversPath)


Function Test-RegistryValue {
    param(
        [Alias("PSPath")]
        [Parameter(Position = 0, Mandatory = $true, ValueFromPipeline = $true, ValueFromPipelineByPropertyName = $true)]
        [String]$Path
        ,
        [Parameter(Position = 1, Mandatory = $true)]
        [String]$Name
        ,
        [Switch]$PassThru
    )

    process {
        if (Test-Path $Path) {
            $Key = Get-Item -LiteralPath $Path
            if ($Key.GetValue($Name, $null) -ne $null) {
                if ($PassThru) {
                    Get-ItemProperty $Path $Name
                } else {
                    $true
                }
            } else {
                $false
            }
        } else {
            $false
        }
    }
}
function test-administrator {
    $identity  = [Security.Principal.WindowsIdentity]::GetCurrent()
    $principal = New-Object Security.Principal.WindowsPrincipal($Identity)
    $principal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
}
function get-currentdirectory {
    $thisName = $MyInvocation.MyCommand.Name
    [IO.Path]::GetDirectoryName((Get-Content function:$thisName).File)
}


if (-not (test-administrator)) {
    write-error @"
You must be running as administrator for this script to function.
Unfortunately, we can't reasonable elevate privileges ourselves
so you need to launch an administrator mode command shell and then
re-run this script yourself.
"@
    exit 1
}


# Lets create directories for WinPE build
foreach ($p in $paths ) {
    if (-not (test-path -path $p)) {
        new-item -type directory $p
    }
}


###
# In order to create a WinPE image we need the ADK.  To automate the process of installing
# the ADk we can use Chocolatey, lets install that now...
###

$result = Test-RegistryValue -Path "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows Kits\Installed Roots\" -Name "KitsRoot81"

if(-not $result) {

    iex ((new-object net.webclient).DownloadString('https://chocolatey.org/install.ps1'))

###
# Install Windows ADK only WinPE requirements
###
    choco install windows-adk-winpe -y
}
# OK where is the ADK?

$KitsRoot81 = (Get-ItemProperty -Path "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows Kits\Installed Roots\" -Name "KitsRoot81").KitsRoot81

$wim = Join-Path $KitsRoot81 $SubPathWinPeImage
Write-Debug $wim

Copy-Item $wim $WimPath

mount-windowsimage -imagepath "$WimPath\winpe.wim" -index 1 -path $MountPath -erroraction stop


foreach ($cab in $PackageCabs ) {
    write-host "** Installing $cab to image"
    # there must be a way to do this without a temporary variable
    $path = "$KitsRoot81$SubPathPackages"

    $pkg = join-path $path "$cab"
    Write-Debug $pkg
    add-windowspackage -packagepath $pkg -path $MountPath
}

foreach ($cab in $LangPackageCabs ) {
    write-host "** Installing $cab to image"
    # there must be a way to do this without a temporary variable
    $path = "$KitsRoot81$SubPathLangPackages"

    $pkg = join-path $path "$cab"
    Write-Debug $pkg
    add-windowspackage -packagepath $pkg -path $MountPath
}


Write-Host "** Installing Drivers to image"
Add-WindowsDriver -Recurse -Path $MountPath -Driver $DriversPath

write-host "* Writing startup PowerShell script"
$file   = join-path $MountPath "hanlon-discover.ps1"
$client = join-path $ScriptPath "hanlon-discover.ps1"
copy-item $client $file

write-host "* Writing Windows\System32\startnet.cmd script"
$file = join-path $MountPath "Windows\System32\startnet.cmd"
set-content $file @"
@echo off
echo Starting wpeinit...
wpeinit
echo Starting Hanlon discover and callbacks...
powershell -executionpolicy bypass -file %SYSTEMDRIVE%\hanlon-discover.ps1
echo dropping to a command shell now...
"@

write-host "* Removing setup.exe from image"
$setup = join-path $MountPath "setup.exe"
Write-Debug $setup
Remove-Item $setup -ErrorAction SilentlyContinue

Write-Host "* Generate Language Files"
dism /image:$MountPath /gen-langini /distribution:$MountPath

write-host "* Unmounting and saving the wim image"

dismount-windowsimage -save -path $MountPath -erroraction stop

Write-Host "* Moving winpe.wim to with date"
$date = get-date -format M-d-yyyy-Hmm

Move-Item "$WimPath\winpe.wim" "$WimPath\winpe-$date.wim"


{% endhighlight %}

# WinPE Hanlon Static Location

Within your static location three files must be present, boot.wim, BCD and boot.sdi.

{% highlight bash %}

wimboot
sources
└── boot.wim
boot
├── bcd
├── BCD
└── boot.sdi


{% endhighlight %}
