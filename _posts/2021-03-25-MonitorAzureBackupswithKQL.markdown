---
title: "Monitoring your Azure Backups with KQL"
excerpt: "Configure Azure backup status with KQL"
#header: 
   #og_image: /assets/images/azspringclean-dine-blog-image.png
    #teaser: /assets/images/azspringclean-dine-blog-image.png
date:   "2021-03-25"
categories: 
- "Automation"
tags: 
- "LogAnalytics"
- "KQL"
- "Azure Backups"
- "Azure Monitoring"
---
Back in the day no one worried about monitoring & reporting, now as companies focus their shift to security, there's a growing interest in
monitoring & reporting.

In this blog post I'll walk you through how I configured Azure an monitoring & dashboard solution for a client scoped to their existing Azure VM's.

In order to configure Azure monitoring, you need to perform the following:
 1. **Analyze the Logs**
 2. **Visualize the logs into user friendly view**

## 1.**Analyze the Logs**

My initial task was to investigate where the logs are stored. Usually we quickly browse to event manager for servers & computers but in Azure we need to enable the below diagnostics on the virtual machine to feed data to "Log Analytics Workspace". 

* **AddonAzureBackupJobs**
* **CoreAzureBackup**

> You can also enable different events on the VM's diagnostic settings based on your desired output. 

Once you've made these changes to the diagnostic settings, allow 24hrs for initial data push to complete. 

Next we need to read the logs which are required for monitoring. We can achieve this by leveraging the power of KQL (Kusto Query language).

## Backup Monitoring Script

I composed this KQL script to monitor the backup status of set of Azure virtual machines. This script mainly relies on the "backupItem" operation to get the desired output.

```powershell

AddonAzureBackupJobs
| where TimeGenerated > ago(24h)
| parse ResourceId with "/"Subscriptions"/"id"/"RG"/"ResourceGroup"/"Provid"/"Services"/"safe"/"RecoveryServiceVault
| where ResourceGroup in ("ResourceGroup1", "ResourceGroup2")
| project ResourceGroup, RecoveryServiceVault, TimeGenerated , BackupResult=JobFailureCode , JobStatus , JobUniqueId , BackupItemUniqueId
| join kind= inner
(
CoreAzureBackup
| where OperationName == "BackupItem"
| project TimeGenerated , VMName=BackupItemFriendlyName , BackupItemUniqueId , ResourceGroupName
)
on BackupItemUniqueId
| project-away BackupItemUniqueId
| project TimeGenerated , VMName , BackupResult , JobStatus , JobUniqueId , ResourceGroup, RecoveryServiceVault
| distinct TimeGenerated , VMName , BackupResult , JobStatus , JobUniqueId , ResourceGroup, RecoveryServiceVault
| sort by TimeGenerated asc nulls first
| summarize Result=count() by BackupResult , VMName , JobStatus , TimeGenerated , ResourceGroup, RecoveryServiceVault
| render piechart

```

In the above script, I'm querying "AddonAzureBackupJobs" event & I'm parsing recovery service vault where the VM's are located in specific resource groups. 

As mentioned, I'm configuring the backup report for a set of virtual machines not for all the virtual machines in Azure. 

Now I'm leveraging join operator to join "AddonAzureBackupJobs" & "CoreAzureBackup" event tables. I'm joining the tabes on "BackupItemUniqueID" to get the unique values of backup results. 

Below is the part of KQL code which performs the operation. 

```powershell

| join kind= inner
(
CoreAzureBackup
| where OperationName == "BackupItem"
| project TimeGenerated , VMName=BackupItemFriendlyName , BackupItemUniqueId , ResourceGroupName
)
on BackupItemUniqueId

```

Once I joined the two event tables, I'm projecting the required outputs "Timegenerated, VMName, Backupresult, jobstatus, jobuniqueId, resourcegroup & recoveryservicevault". 

After projecting the desired outputs, I'm marking the outputs based on "timegenerated", I'm able to achieve this by using the "distinct" operator in KQL. 

```powershell

| distinct TimeGenerated , VMName , BackupResult , JobStatus , JobUniqueId , ResourceGroup, RecoveryServiceVault

```

## Visualize the logs into user friendly view

After getting the desired output, I want to present the results to stakeholders & operation managers. 

In order to convert the technical ouput into a user friendly view I chose to use PieChart. You can use different visualization methods it all depends on how stakeholders want to see the output. 

So, to setup a piechart displaying how many VM's are successfully backed up. I used the following PowerShell:

```powershell

| summarize Result=count() by BackupResult , VMName , JobStatus , TimeGenerated , ResourceGroup, RecoveryServiceVault
| render piechart

```

Once everyone is satisfied with the result, you can always pin your resulting piechart of other visualization to the Azure dashboard, so rather than executing the KQL script above everytime you can simply check Azure dashboard for the results.

Chiru