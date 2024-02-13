---
layout: post
title: Retrieving a List of Installed Applications on Multiple Citrix VDIs with PowerShell
category:
- Scripting
- PowerShell
tags:
- citrix
date: 2024-02-01 19:20 -0700
---
In this blog post, we'll explore a PowerShell script that retrieves a list of installed applications on multiple Citrix virtual desktops. This script leverages the Citrix PowerShell module to gather information about virtual desktops and utilizes WMI (Windows Management Instrumentation) to retrieve details about installed applications.

## The Script Breakdown

Let's break down the key components of the script:

1. **Loading the Citrix Module**: The script begins by loading the Citrix PowerShell module using the `asnp citrix*` command. This module provides cmdlets for managing Citrix environments.

2. **Defining the `Get-InstalledApplications` Function**: The script defines a function named `Get-InstalledApplications`, which takes the computer name as an input parameter. Inside this function, the `Get-WmiObject` cmdlet is used to query the Win32_Product class, retrieving details about installed applications such as name, version, and vendor. The list of applications is then sorted alphabetically by name.

3. **Array of Desktop IDs**: An array named `$DesktopId` is defined, containing the Broker Desktop Group UUIDs of the Citrix virtual desktops. These IDs are used to identify and retrieve information about the virtual desktops.

4. **Retrieving Virtual Desktop Information**: The script uses a foreach loop to iterate over each Desktop ID in the `$DesktopId` array. Within the loop, the `Get-BrokerMachine` cmdlet is used to obtain information about each virtual desktop, specifically the DNS name. This information is stored in an array named `$VdiArray`.

5. **Retrieving Installed Applications**: Another foreach loop is used to iterate over each virtual desktop in `$VdiArray`. For each virtual desktop, the `Get-InstalledApplications` function is called with the DNS name of the desktop as an argument. This function retrieves the list of installed applications, and the results are output to the console.

> Before implementing any script or code in a production environment, always perform thorough testing in a controlled and non-critical setting. Understand that errors or unexpected behavior may occur, so validate and adapt the code to your specific use case.
{: .prompt-info }

## Script

```powershell
# Load the Citrix module
asnp citrix*

# Define a function to get installed applications on a computer
function Get-InstalledApplications ($ComputerName) {
    # Use the WMI class Win32_Product to get a list of installed applications
    # Select the Name, Version, and Vendor properties
    # Sort the results by the Name property
    return Get-WmiObject -Class Win32_Product -ComputerName $ComputerName | Select-Object Name, Version, Vendor | Sort-Object Name
}

# Define an array of Desktop IDs
# Replace $DesktopId with the Broker Desktop Group UUID of your Citrix machines. Multiple UUIDs maybe be included at once
$DesktopId = "0000", "0000", "0000", "0000"

# Create an empty array to store the VDI objects
$VdiArray = @()

# Loop through each Desktop ID
# Replace the -AdminAddress with the name of the primary Citrix Delivery Controller
foreach ($id in $DesktopId) {
    # Get the VDI object with the specified Desktop Group UUID
    # Select the DNSName property
    $VdiObject = Get-BrokerMachine -DesktopGroupUUID $id -AdminAddress testdc01 | Select DNSName
    # Add the VDI object to the array
    $VdiArray += $VdiObject
}

# Loop through each VDI in the array
foreach ($vdi in $VdiArray) {
    # Get the installed applications on the VDI using the Get-InstalledApplications function
    $Applications = Get-InstalledApplications -ComputerName $vdi.DNSName
    # Write the output to the console
    Write-Output "$($vdi.DNSName) has the following applications installed:`n"
    Write-Output $Applications
    Write-Output "`n"
} 
```