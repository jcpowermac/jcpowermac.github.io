---
layout: post
title: Hanlon now supports Windows provisioning!
---

It has been a long road but we finally have the ability to provision Windows in Hanlon.  Currently it is limited to Windows 2012 R2 but the foundation is available to create additional models for the various client and server versions.  The rest of the post is devoted to the Windows specific aspects of Hanlon provisioning a Windows instance, Tom McSweeney has created an excellent [blog post](https://osclouds.wordpress.com/?p=71) describing the server component changes

# Design Details

One of the important goals of this effort was to make sure that Windows provisioning followed as close as possible to the design of existing OS images, models and policies.  This with the added constraint of avoiding a CIFS share if at all possible provides an incredibly unique solution for Windows installation.

## WinPE Image

Since building a WinPE image is required we tried to make it as painless and automated as possible.  We have included the `build-winpe.ps1` script within `${HANLON}/scripts/winpe`.

### Image Details
We add quite a few packages to the WinPE image, this includes:

- WinPE-WMI
- WinPE-NetFx
- WinPE-Scripting
- WinPE-PowerShell
- WinPE-Setup
- WinPE-Setup-Server
- WinPE-DismCmdlets

There are a few to note directly.  We used PowerShell extensively in the hanlon-discovery and windows_install.erb, PowerShell is significantly easier to manage than legacy scripting languages, so WinPE-PowerShell is included.  WinPE-Setup-Server as named is the setup executable and files required to launch a Server based setup, which we need since we are not using CIFS mapped back to an extracted Windows ISO.  And finally including WinPE-DismCmdlets is our solution to adding drivers to the installed Windows image.

### Drivers

To support your specific hardware at a minimum network and storage drivers will be required and available in `%SYSTEMDRIVE%\drivers` at the time of `build-winpe.ps1` execution.  The directory structure should not matter since we are using the recursive option with `Add-WindowsDriver` cmdlet.

### Build Process

The build script performs the following actions:

- First, it creates a mount point for WinPE, a directory for the scripts we're adding to WinPE, and a directory for the drivers that are being added to WinPE if it doesn't already exist (these directories are created on the root of the system drive).
- Next, it tests for an installation of [chocolatey](https://chocolatey.org/) and installs if unavailable.
- Then install [Windows ADK](https://msdn.microsoft.com/en-us/library/hh825420.aspx) via chocolatey.  If this fails, execute our fall back method for installation.
- Next, it copies the the winpe image (`winpe.wim`) from Windows ADK path to a temporary location and mounts that image locally.
- It then adds a few additional (required) Windows packages to the winpe image
- Next, it adds an English language pack to the winpe image
- Then, it adds the drivers needed for network and storage in your hardware to the winpe image (from the `C:\Drivers` folder)
- Next, it downloads the `hanlon-discover.ps1` script from Hanlon's GitHub repository and adds that script to the winpe image
- Next, it creates a `startnet.cmd` script and adds it to the WinPE image, generates a language ini file, and modifies the winpe registry (to disable the CD-ROM when the winpe image boots).
- Finally, it unmounts the winpe image and renames the resulting image based on date and time.

### Execution of build-winpe.ps1

Open a new PowerShell prompt and execute the commands below.  The saved image will be in `%SYSTEMDRIVE%\winpe` named based on the current date and time.

{% highlight PowerShell %}
Set-ExecutionPolicy Bypass -Confirm:$false -Force
Invoke-WebRequest -Uri 'https://raw.githubusercontent.com/csc/Hanlon/master/scripts/winpe/build-winpe.ps1' -OutFile build-winpe.ps1
. .\build-winpe.ps1
{% endhighlight %}

### Video Demo of Build Process
<iframe width="560" height="315" src="https://www.youtube.com/embed/aRKbiwEIJYw" frameborder="0" allowfullscreen></iframe>


## Hanlon Discover
The only third-party modification to WinPE image is the addition of the PowerShell script `hanlon-discover.ps1`.  The only purpose of this script is to determine the location of the Hanlon server and execute the `windows_install.erb`.

We discovered last April that Windows DHCP client has no easy mechanism to support user-defined options without using the Win32 APIs.  Fortunately we ran across this [blog post](http://www.ingmarverheij.com/read-dhcp-options-received-by-the-client/) providing the necessary details and PowerShell script that we could start with to retrieve the Hanlon specific options from the DHCP server.  The Windows DHCP client supports `vendor-encapsulated-options` which can be used in conjunction with `vendor-option-space` to construct the encapsulated option.

Creating a option space:

{% highlight bash %}
option space hanlon;
option hanlon.server code 224 = ip-address;
option hanlon.port code 225 = unsigned integer 16;
option hanlon.base_uri code 226 = text;
{% endhighlight %}

Using within a subnet definition:

{% highlight bash %}
class "MSFT" {
  match if substring (option vendor-class-identifier, 0, 4) = "MSFT";
  option hanlon.server 192.168.122.254;
        option hanlon.port 8026;
        option hanlon.base_uri "/hanlon/api/v1";
        vendor-option-space hanlon;
}
{% endhighlight %}

We create an object based on the values retrieved from the `vendor-encapsulated-options` section of the registry.  The object properties are used to construct the proper uri to the RESTful endpoint of Hanlon.  Since there isn't a method to pass boot time arguments to Windows we use WMI to determine the SMBIOS uuid which can be used in a RESTful request to retrieve the `active_model`.  Once the active_model is determined the `windows_install.erb` is requested and executed which continues the process.  

## Windows Install
This script picks up where `hanlon-discover.ps1` leaves off.  The first task is where to grab the `install.wim`.  In order to deploy Windows without CIFS we needed a temporary location to store the `install.wim`, but the question was where to put it.  I knew that Windows setup would want the beginning of the disk for itself so why not create a partition at the end of the disk.  Using WMI to grab the size of the disk, we subtract 8GB and use that as the offset to diskpart.

{% highlight PowerShell %}

$image_disk_size = 8
$offset = [math]::floor((Get-WmiObject Win32_DiskDrive).size / 1024) - (1024*1024*$image_disk_size)

$command = @"
select disk 0
clean
create partition primary offset=$offset
select volume 0
format fs=ntfs label=image quick
assign letter=I
"@

$command | diskpart
{% endhighlight %}

I am sure some of you are wondering can I get that 8GB returned and the answer is yes.  Just delete the partition and extend the system drive within Disk Management.  In a follow up we will be adding additional scripts that could perform this action.

Next we download the drivers, the `install.wim` and the `autounattend.erb`.  After those three files are downloaded we inject the required drivers into the `install.wim` using the DISM PowerShell cmdlets.

{% highlight PowerShell %}
mount-windowsimage -imagepath "I:\install.wim" -index <%= wim_index %> -path "I:\mount-point" -erroraction stop
Add-WindowsDriver -Recurse -Path "I:\mount-point" -Driver "I:\drivers"
dismount-windowsimage -save -path "I:\mount-point" -erroraction stop
{% endhighlight %}

Therefore we don't force a user of Hanlon to modify their `install.wim` in anyway but can still provide the ability to install on hardware that may require specific drivers.  This also means potentially additional packages or updates could be injected to the image if desired.

Once setup is complete there are a series of callback to Hanlon and finally the WinPE instance reboots.

## Auto Unattend

The `autounattend.erb` is very simple as an `unattend.xml` usually is.  I think there is only a few things to point out.  Since we are creating a partition at the end of the disk I need to make sure `WillWipeDisk` is set to false

{% highlight xml %}
<WillWipeDisk>false</WillWipeDisk>
{% endhighlight %}

To determine which edition of Windows to install and core vs GUI we use the following option:
{% highlight xml %}
<MetaData>
<Key>/IMAGE/INDEX</Key>
<Value><%= wim_index %></Value>
</MetaData>

{% endhighlight %}

And finally our last callback we use the RunSynchronousCommand and PowerShell.

{% highlight xml %}
<component name="Microsoft-Windows-Deployment" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS" xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <RunSynchronous>
                <RunSynchronousCommand wcm:action="add">
                    <Order>1</Order>
                    <Path>powershell -NoLogo -Command "Invoke-WebRequest -Uri  <%= callback_url("postinstall", "final") %> -Method GET -UseBasicParsing"</Path>
                    <Description>Hanlon Postinstall Final</Description>
                </RunSynchronousCommand>
            </RunSynchronous>
      </component>
{% endhighlight %}


# Conclusion
IMHO we have created a one of a kind, most automated installation mechanism for deploying Windows instances.  Without requiring a CIFS and the ability to inject drivers on the fly to the `install.wim` we have reduced complexity in deployment.  

# Acknowledgements

**Tom McSweeney**

Without Tom we certainly not be here today.  He did extensive changes to the image slice, model slice, static location and modified the `app.rb` to support chunking significantly reducing memory requirements.  It has been educating experience and I am glad that I got the opportunity to work with him on this project.

**Chris Yentha**

After struggling with the `unattend.xml` for hours, a single email to Chris provided us a working example that we could use in the `autounattend.erb`.

**Russell Teague and Aaron Dean**

Both Russell and Aaron provided moral support and offloading of tasks to allow us to continue with this initiative.  
