---
layout: post
title: Inactive Citrix VDI Resource Email with PowerShell
date: 2024-02-13 16:08 -0700
category:
- Scripting
- PowerShell
tags:
- citrix
- vdi
---
Managing Citrix Virtual Desktop Infrastructure (VDI) efficiently is crucial for IT administrators. A common challenge in this area is identifying inactive users who unnecessarily consume resources. To address this issue, I've developed a PowerShell script to automate the detection of inactive VDI sessions, optimizing resource usage.

## Script Breakdown
This script is customizable to fit the specific needs of your Citrix environment. Here's how you can tailor it:

- **Date Calculation:** The script currently checks for resources inactive for the past three months (`AddMonths(-3)` on line 8). Adjust this period to match your policy by changing the number of months, such as -2 for two months or -6 for six months, depending on your operational requirements.

- **Target Citrix DeliveryController:** Update the `Admin-Address` parameter on line 17 with the address of your Delivery Controller to ensure the script targets the right environment.

- **CSS Styles:** The email report's appearance can be fully customized with CSS. This flexibility allows you to match the report's styling to your organizational branding or improve its readability. Feel free to adjust the CSS styles as needed.

- **Email Parameters:** The script includes parameters for sending an email report. Update these parameters, starting on line 64, with your actual SMTP server details and the appropriate sender and recipient email addresses to ensure the report reaches the right stakeholders.

## Running the Script
Deploying this script can be achieved by setting up a Scheduled Task on a Windows Server. This approach allows for the automation to run at regular intervals, such as monthly, ensuring consistent monitoring and reporting of inactive VDI sessions. Here's a brief guide on setting up a Scheduled Task:

1. Open Task Scheduler on your Windows Server.
2. Create a new task and configure the trigger to your preferred schedule, for example, monthly.
3. Under the Actions tab, set the action to "Start a program" and browse to the PowerShell executable.
4. In the "Add arguments" section, input the path to your script file.
5. Ensure the task runs with the appropriate permissions and has access to the necessary resources to execute the script.

> Before implementing any script or code in a production environment, always perform thorough testing in a controlled and non-critical setting. Understand that errors or unexpected behavior may occur, so validate and adapt the code to your specific use case.
{: .prompt-info }

## Script
```powershell
# Load Citrix snap-ins
asnp Citrix*

# Multiple Citrix Desktop IDs can be entered
$desktopid = "0000", "0000", "0000"

# Calculate the date three months prior to today to identify inactive VDIs
$Lastused =((Get-Date).AddMonths(-3).ToString('yyyy-MM-dd HH:mm:s'))

# Initialize an empty array to store user details who match the criteria
$users = @()

# Loop through each Desktop ID to find and add inactive users to the $users array
foreach ($id in $desktopid) {

    # Replace TESTDC1 with your Delivery Controller
    $user = Get-BrokerMachine -DesktopGroupUUID $id -AdminAddress TESTDC1 |
    Where-Object {$_.LastConnectionTime -lt $Lastused -and $_.LastConnectionTime -gt "1999-12-30 00:00:00" -and $_.InMaintenanceMode -match "False" -and $_.SessionCount -lt "1" }  |
    Select-Object DNSName,LastConnectionTime,{$_.AssociatedUserNames}

    # Add the results to the $users array
    $users += $user

}

# Sort data by last connection time and convert it to HTML format for the email
$usersOutput = $users | Sort-Object LastConnectionTime | ConvertTo-Html

# Define CSS styles for the HTML email
$css =
@"

<Style>

    table { border-collapse: collapse; font-family: sans-serif; font-size: 0.85em; min-width: 800px; }

    th, td { border: 1px solid #E3E3E3; text-align: left; }

    th { background: #4d6466; color: #fff; padding: 5px 10px; }

    td { color: #000; padding: 5px 10px; }

    tr { background: #fff; }

</Style>

"@

# Compile the HTML email content
$emailMessage =
@"

$css

<p style="font-family: sans-serif; font-size: 1.25em;">Inactive VDIs - No Login 90+ Days</p>

$usersOutput

<p style="font-family: sans-serif">Report Date: $(Get-Date)</p>

"@

# Set up email parameters
$emailParam = @{
    from       = "VDICheck@something.com"
    to         = "hi@something.com"
    subject    = "Inactive VDIs - No Login 90+ Days"
    body       = $emailMessage
    bodyashtml = $true
    smtpserver = "smtp.something.com"
    port       = 25
}

# Send the email report
Send-MailMessage @emailParam
```