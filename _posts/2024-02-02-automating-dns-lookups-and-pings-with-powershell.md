---
layout: post
title: Automating DNS Lookups and Pings with PowerShell
category:
- Scripting
- PowerShell
tags:
- nslookup
- ping
date: 2024-02-02 13:26 -0700
---
In this article, we will explore a PowerShell script that reads a list of hostnames or IP addresses from a file, performs DNS lookups, and then pings each address, outputting the results in a user-friendly manner.

## The Script Breakdown

The script is designed to be both simple and effective, making it accessible to PowerShell beginners while being a useful tool for experienced administrators. Here's a step-by-step breakdown:

### Setting Up the Input File

The script starts by defining a parameter, `$InputFile`, which specifies the path to a text file containing a list of hostnames or IP addresses, one per line. This file is the script's input, and its location must be provided when the script is run.

### Verifying the Input File

Before proceeding, the script checks if the specified input file exists using the `Test-Path` cmdlet. If the file is not found, the script outputs an error message and exits, preventing any attempts to process a non-existent file.

### Reading and Processing Each Address

The script reads the input file into an array, `$addresses`, with each line (address) becoming an element of the array. It then iterates over each address, performing the following actions:

1. **DNS Lookup**: For each address, the script attempts a DNS lookup using the `Resolve-DnsName` cmdlet. If successful, it outputs the resolved hostnames and IP addresses. If the lookup fails, an error message is displayed.

2. **Ping**: Next, the script attempts to ping the address using the `Test-Connection` cmdlet, configured to send just one ping (`-Count 1`). If the ping is successful, the script outputs a success message along with the response time. If the ping fails, an error message is displayed.

## Running the Script

To use the script, save it as a `.ps1` file, and run it from PowerShell, passing the path to your input file as the `InputFile` parameter:

```powershell
.\YourScriptName.ps1 -InputFile "C:\path\to\your\list.txt"
```
> Before implementing any script or code in a production environment, always perform thorough testing in a controlled and non-critical setting. Understand that errors or unexpected behavior may occur, so validate and adapt the code to your specific use case.
{: .prompt-info }

## Script

```powershell
# Define a parameter for the script to take an input file path as an argument.
param(
    [string]$InputFile
)

# Check if the specified input file exists.
if (-not (Test-Path $InputFile)) {
    # If the file does not exist, output an error and exit the script.
    Write-Error "Input file not found."
    exit
}

# Read the hostnames or IP addresses from the input file into an array.
$addresses = Get-Content $InputFile

# Iterate over each address to perform DNS lookup.
Write-Host ""
foreach ($address in $addresses) {
    # Notify the user which address is being processed.
    Write-Host "Attempting to resolve: $address"
    # Run NSLookup
    try {
        # Attempt to resolve the hostname or IP address.
        $nsresults = Resolve-DnsName $address -ErrorAction Stop
        # Output the resolved hostnames and IP addresses.
        foreach ($nsresult in $nsresults) {
            Write-Host "$address NSLookup: $($nsresult.NameHost)"
        }
    }
    catch {
        # If the resolution fails, output an error message.
        Write-Host "$address NSLookup: Failed" -ForegroundColor Red
    }
    # Run Ping
    try {
        # Attempt to ping the hostname or IP address.
        $pingresults = Test-Connection -ComputerName $address -Count 1 -ErrorAction Stop   
        # Output the pinged hostnames and IP addresses.
        foreach ($pingresult in $pingresults) {
            $responseTime = $pingresult.ResponseTime
            Write-Host "$address Ping: Success (Response time: $responseTime ms)"
        }
    }
    catch {
        # If the ping fails, output an error message.
        Write-Host "$address Ping: Failed" -ForegroundColor Red
    }    
        
    # Add an empty line for spacing between results.
    Write-Host ""
}
```
