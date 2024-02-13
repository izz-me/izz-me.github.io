---
layout: post
title: Creating a PowerShell-based GUI for Managing Citrix MCS Images
category:
- Scripting
- PowerShell
tags:
- citrix
- MCS
date: 2024-02-7 14:54 -0700
---
Updating Citrix Machine Creation Services images is a pain. It can only be done via PowerShell. This script attempts (and I think succeeds) to make creating that script simpler. I was going away for a while and needed to leave something in case the Junior Admin got in trouble. A GUI seemed appropriate. Sorry, it's ugly. Welcome to 1995.

## Key Features

- **Fetching Provisioning Schemes:** Connects to the Citrix Delivery Controller to retrieve and display current provisioning schemes. As such, make sure the scripts is run with appropriate Citrix domain rights.
- **VM Snapshots Management:** Enables the selection of provisioning schemes to fetch and display associated VM snapshots. The script only searches for snapshots in the original VM used as a target for the image. In other words, if a totally new VM is created then you will need to manually write the image update script.
- **Script Generation:** Facilitates generating PowerShell scripts to publish updated master VM images based on selected provisioning schemes and snapshots.

## How It Works

1. **Preparation:** The script begins by loading the required .NET Framework assemblies for Windows Forms and data handling, and it adds Citrix PowerShell snap-ins to access Citrix cmdlets.

2. **GUI Creation:** A form is created with a user-friendly layout, including buttons for fetching data, text boxes for status messages, and DataGridViews for displaying the provisioning schemes and snapshots.

3. **Fetching Data:** Upon clicking the "Fetch Provisioning Scheme" button, the script prompts for the Citrix Delivery Controller's address, retrieves provisioning schemes, and populates a data table. This data is then displayed in a DataGridView.

4. **Snapshot Management:** Another button, initially disabled, allows for fetching VM snapshots for a selected provisioning scheme. It becomes enabled once a provisioning scheme is selected.

## Important Note on Citrix Delivery Controller Integration

Don't forget to add your own Delivery Controller in line 23:

```powershell
$provisioningSchemes = Get-ProvScheme -AdminAddress TESTDC1
```
{: .nolineno }

> Before implementing any script or code in a production environment, always perform thorough testing in a controlled and non-critical setting. Understand that errors or unexpected behavior may occur, so validate and adapt the code to your specific use case.
{: .prompt-info }

## Script

```powershell
Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Data
# Load the Citrix modules
Add-PSSnapIn citrix*

# Define the form
$form = New-Object System.Windows.Forms.Form
$form.Text = "Fetch MCS Catalog Image "
$form.Size = New-Object System.Drawing.Size(1300, 1000)

# Set a light blue background color for the form
$form.BackColor = [System.Drawing.Color]::LightBlue

# Add a Fetch button
$button = New-Object System.Windows.Forms.Button
$button.Location = New-Object System.Drawing.Point(10, 10)
$button.Size = New-Object System.Drawing.Size(250, 40)
$button.Text = "Fetch Provisioning Scheme"
$button.Add_Click({
    $fetchStatusBox.Text += "Fetching data... "

    # Replace TESTDC1 with the address of your Citrix Delivery Controller
    $provisioningSchemes = Get-ProvScheme -AdminAddress TESTDC1

    $fetchStatusBox.Text += "> Data fetched"

    # Clear existing rows in the DataTable
    $dataTable.Rows.Clear()

    # Clear existing rows in the Snapshot DataTable
    $spDataTable.Rows.Clear()

    # Populate the DataTable with the fetched data
    $provisioningSchemes | ForEach-Object {
        $row = $dataTable.NewRow()
        $row["ProvisioningSchemeName"] = $_.ProvisioningSchemeName
        $row["ProvisioningSchemeUid"] = $_.ProvisioningSchemeUid
        $row["HostingUnitName"] = $_.HostingUnitName
        $row["MasterImageVM"] = $_.MasterImageVM
        $row["MasterImageVMDate"] = $_.MasterImageVMDate
        $row["MachineCount"] = $_.MachineCount
        $dataTable.Rows.Add($row)
    }

    # Auto-size the columns
    $dataGridView.AutoResizeColumns([System.Windows.Forms.DataGridViewAutoSizeColumnsMode]::AllCells)

    $fetchStatusBox.Text += "> Data displayed " + "`r`n"
})

$form.Controls.Add($button)

# Add a TextBox for fetch status
$fetchStatusBox = New-Object System.Windows.Forms.TextBox
$fetchStatusBox.Multiline = $true
$fetchStatusBox.ScrollBars = "Vertical"
$fetchStatusBox.Location = New-Object System.Drawing.Point(270, 10)
$fetchStatusBox.Size = New-Object System.Drawing.Size(600, 40)
$fetchStatusBox.ReadOnly = $true
$form.Controls.Add($fetchStatusBox)

# Create a DataTable
$dataTable = New-Object System.Data.DataTable
$dataTable.Columns.Add("ProvisioningSchemeName")
$dataTable.Columns.Add("ProvisioningSchemeUid")
$dataTable.Columns.Add("HostingUnitName")
$dataTable.Columns.Add("MasterImageVM")
$dataTable.Columns.Add("MasterImageVMDate")
$dataTable.Columns.Add("MachineCount")

# Add a DataGridView
$dataGridView = New-Object System.Windows.Forms.DataGridView
$dataGridView.Location = New-Object System.Drawing.Point(10, 60)
$dataGridView.Size = New-Object System.Drawing.Size(1250, 300)
$dataGridView.BackgroundColor = [System.Drawing.Color]::White
$dataGridView.DataSource = $dataTable

# Set the DataGridView properties
$dataGridView.SelectionMode = [System.Windows.Forms.DataGridViewSelectionMode]::FullRowSelect
$dataGridView.ReadOnly = $true

$form.Controls.Add($dataGridView)

# Add a Fetch VM Snapshots button (initially disabled)
$fetchSnapshotsButton = New-Object System.Windows.Forms.Button
$fetchSnapshotsButton.Location = New-Object System.Drawing.Point(10, 380)
$fetchSnapshotsButton.Size = New-Object System.Drawing.Size(250, 40)
$fetchSnapshotsButton.Text = "Fetch VM Snapshots"
$fetchSnapshotsButton.Enabled = $false  # Disable initially
$fetchSnapshotsButton.Add_Click({
    # Get the selected row
    $selectedRow = $dataGridView.SelectedRows[0].DataBoundItem

    # Extract %hostname% and %vmName% from MasterImageVM
    $masterImageVM = $selectedRow.MasterImageVM
    $hostname = ($masterImageVM -split '\\')[2]
    $vmName = ($masterImageVM -split '\\')[3]

    # Build the path and run the Get-ChildItem command
    $path = "XDHyp:\hostingunits\$hostname\$vmName"
    $snapshots = Get-ChildItem -Recurse -Path $path

    # Clear existing rows in the Snapshot DataTable
    $spDataTable.Rows.Clear()

    # Capture the value of $vmName in the outer scope
    $currentVMName = $vmName

    # Populate the Snapshot DataTable with the fetched data
    $snapshots | ForEach-Object {
    $row = $spDataTable.NewRow()
    $row["BaseVMName"] = $currentVMName
    $row["FullName"] = $_.FullName
    $row["FullPath"] = $_.FullPath
    $spDataTable.Rows.Add($row)
    }

    # Auto-size the columns
    $spDataGridView.AutoResizeColumns([System.Windows.Forms.DataGridViewAutoSizeColumnsMode]::AllCells)
})

$form.Controls.Add($fetchSnapshotsButton)

# Add a selection changed event for the Provisioning Scheme DataGridView
$dataGridView.Add_SelectionChanged({
    # Enable or disable the "Fetch VM Snapshots" button based on whether a row is selected
    if ($dataGridView.SelectedRows.Count -gt 0) {
        $fetchSnapshotsButton.Enabled = $true
    } else {
        $fetchSnapshotsButton.Enabled = $false
    }
})

$form.Controls.Add($fetchSnapshotsButton)

# Create a Snapshot DataTable
$spDataTable = New-Object System.Data.DataTable
$spDataTable.Columns.Add("BaseVMName")
$spDataTable.Columns.Add("FullName")
$spDataTable.Columns.Add("FullPath")

# Add a Snapshot DataGridView
$spDataGridView = New-Object System.Windows.Forms.DataGridView
$spDataGridView.Location = New-Object System.Drawing.Point(10, 430)
$spDataGridView.Size = New-Object System.Drawing.Size(1250, 300)
$spDataGridView.BackgroundColor = [System.Drawing.Color]::White  # Set background color to white
$spDataGridView.DataSource = $spDataTable

# Set the Snapshot DataGridView properties
$spDataGridView.SelectionMode = [System.Windows.Forms.DataGridViewSelectionMode]::FullRowSelect
$spDataGridView.ReadOnly = $true

$form.Controls.Add($spDataGridView)

# Add a Generate Script button (initially disabled)
$generateScriptButton = New-Object System.Windows.Forms.Button
$generateScriptButton.Location = New-Object System.Drawing.Point(10, 750)
$generateScriptButton.Size = New-Object System.Drawing.Size(250, 40)
$generateScriptButton.Text = "Generate Script"
$generateScriptButton.Enabled = $false  # Disable initially
$generateScriptButton.Add_Click({
    # Get the selected rows
    $selectedProvSchemeRow = $dataGridView.SelectedRows[0].DataBoundItem
    $selectedSnapshotRow = $spDataGridView.SelectedRows[0].DataBoundItem

    # Extract data from selected rows
    $provisioningSchemeName = $selectedProvSchemeRow.ProvisioningSchemeName
    $fullPath = $selectedSnapshotRow.FullPath

    # Generate the script
    $script = @"
Add-PSSnapIn citrix*
Publish-ProvMasterVmImage -ProvisioningSchemeName "$provisioningSchemeName" -MasterImageVM "$fullPath"
"@
    
    # Display the script in the TextBox
    $generatedScriptTextBox.Text = $script
})

$form.Controls.Add($generateScriptButton)

# Add a TextBox for displaying the generated script
$generatedScriptTextBox = New-Object System.Windows.Forms.TextBox
$generatedScriptTextBox.Multiline = $true
$generatedScriptTextBox.ScrollBars = "Vertical"
$generatedScriptTextBox.Location = New-Object System.Drawing.Point(10, 800)
$generatedScriptTextBox.Size = New-Object System.Drawing.Size(1250, 100)
$generatedScriptTextBox.ReadOnly = $true
$form.Controls.Add($generatedScriptTextBox)

# Add a selection changed event for the Snapshot DataGridView
$spDataGridView.Add_SelectionChanged({
    # Enable or disable the "Generate Script" button based on whether both a Provisioning Scheme and a VM Snapshot are selected
    if ($dataGridView.SelectedRows.Count -gt 0 -and $spDataGridView.SelectedRows.Count -gt 0) {
        $generateScriptButton.Enabled = $true
    } else {
        $generateScriptButton.Enabled = $false
    }
})

# Show the form
$form.ShowDialog()
```