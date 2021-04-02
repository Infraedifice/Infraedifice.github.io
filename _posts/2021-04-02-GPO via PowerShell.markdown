---
title: "GPO report on Active Directory OU with PowerShell"
excerpt: "Extract GPO report on AD OU's with PowerShell"
header: 
    og_image: /assets/images/GPOreporton ADOUwithPowerShell.png
    teaser: /assets/images/GPOreporton ADOUwithPowerShell.png
date:   "2021-04-01"
categories: 
- "Automation"
tags: 
- "Group Policy Object"
- "PowerShell"
- "Active Directory"

---
Hello readers, in this blog I'm going to explain how to identify which GPO (Group Policy Object) is applied on an AD OU. 

In my experience with large organizations where Active Directory contains hundreds of OU (Organization Units) & whenever an IT admin wants to access the GPO report, they need to access the group policy management console to check which  GPO is applied on what OU.

One of my clients approached me to get the report of existing GPO's so that they can analyze the stale GPO & remediate them. You can get that by using Get-Gporeport from Microsoft docs but the report is in either HTML or XML & the client wants to know which GPO is applied on what OU.
My client wants to read it in a quick user-friendly way (excel report), so I decided to export the GPO report using PowerShell. The way I did it is by querying each OU in Active Directory & check which GPO is applied on it. 
So, all you need is an Active Directory administrator access & AD module installed in PowerShell. This script provides the following. 

1. **Display Name of GPO**
2. **GPO enabled on OU**
3. **GPO enforced on OU**

By using the above 3 you should be able to report if a GPO is applied on an AD OU or not. Let's dive into the PowerShell script. 

``` powershell
# This script uses a function to check the AD module installation and loads it if it's not loaded already. 
Function Check-ADModule
{
  Try
  {
    $check = Get-Addomain
  }
  catch
  {
  $fault = $error[0].exception.message
  }
 if ($check)
 {
  Write-Host -ForegroundColor Cyan "Active Directory module loaded"
 }
 else
 {
  Write-Host -ForegroundColor red "We need to Import Active Directory Module"
  Import-Module ActiveDirectory
  # The above command imports AD module.
 }
}
# This Script Gives information about all the GPO's associated with each OU in the domain. 
# In this script we get the OU information first and then we will get the GPO links for each of them.
Check-ADModule
# The above Function checks for AD Module Installation.  
$OU = Get-adorganizationalunit -filter 'Name -like "*"' | Select Name, DistinguishedName
$i = 1
$count = $ou.count
$a = @()
Foreach ($obj in $OU)
{
Write-Progress -Activity "Getting GPO information on all the $($obj.Name)" -Status "$i of $count" -PercentComplete ($i/$count*100)
    $gpo = (Get-ADOrganizationalUnit -Identity $($obj.DistinguishedName) | Get-GPInheritance).Gpolinks | Select GPOID, DisplayName, Enabled, Enforced, Target, Order
      Foreach ($GP in $gpo)
        {
            $result = New-Object psobject 
            $result | Add-Member -Type NoteProperty -Name "OU Name" -Value $($obj.Name)
            $result | Add-Member -Type NoteProperty -Name "DistinguishedName" -Value $($obj.DistinguishedName)
            $result | Add-Member -Type NoteProperty -Name "GPODisplayName" -Value $gp.DisplayName
            $result | Add-Member -Type NoteProperty -Name "GPOID" -Value $gp.GpoId
            $result | Add-Member -Type NoteProperty -Name "Target" -Value $gp.Target
            $result | Add-Member -Type NoteProperty -Name "Enabled" -Value $gp.Enabled
            $result | Add-Member -Type NoteProperty -Name "Enforced" -Value $gp.Enforced
            $result | Add-Member -Type NoteProperty -Name "Order" -Value $gp.Order
            $a += $result
        }
        # End of Foreach loop for GPO. 
    $i++
} # End of script.
$filepath = "GPOresults" + (Get-date -Format "dd-MM-yyyy HH:mm:ss") + ".Csv"
# The above command creates a filepath and name to store the GP Results. 
$a | Export-Csv -Path $filepath -NoTypeInformation
```
Above is a simple script that will query the AD organization units for GPO (Group Policy Objects) linked on the OU's or enforced on them. The script comprises 3 parts. 

1. **Module Check**
2. **Main Script**
3. **Output**

Let's talk about each of them.

## **1. Module Check**

Most of the people run the script without checking for pre-requisite modules. I've done that in the initial days of my career. So I thought it would be good to include a function that will check if the required module is installed or not. The below function achieves it by running "Get-ADdomain" cmdlet which is loaded to a variable. If the cmdlet retrieves the result then AD module is installed if not "Import-Module Active Directory" cmdlet will do it for you. 

```powershell
Function Check-ADModule
{
  Try
  {
    $check = Get-Addomain
  }
  catch
  {
  $fault = $error[0].exception.message
  }
 if ($check)
 {
  Write-Host -ForegroundColor Cyan "Active Directory module loaded"
 }
 else
 {
  Write-Host -ForegroundColor red "We need to Import Active Directory Module"
  Import-Module ActiveDirectory
  # The above command imports the AD module.
 }
}
# This Script Gives information about all the GPO's associated with each OU in the domain. 
# In this script we get the OU information first and then we will get the GPO links for each of them.
Check-ADModule
```
## **2. Main Script**

This is where the action takes place, in here script will query all the AD Ou's using "Get-ADOrganizationunit" cmdlet & the script will loop into each of the OU to query GPO. I specified an empty array "a" to store the output.

```powershell
$OU = Get-adorganizationalunit -filter 'Name -like "*"' | Select Name, DistinguishedName
$i = 1
$count = $ou.count
$a = @()
Foreach ($obj in $OU)
{
Write-Progress -Activity "Getting GPO information on all the $($obj.Name)" -Status "$i of $count" -PercentComplete ($i/$count*100)
    $gpo = (Get-ADOrganizationalUnit -Identity $($obj.DistinguishedName) | Get-GPInheritance).Gpolinks | Select GPOID, DisplayName, Enabled, Enforced, Target, Order
      Foreach ($GP in $gpo)
        {
            $result = New-Object psobject 
            $result | Add-Member -Type NoteProperty -Name "OU Name" -Value $($obj.Name)
            $result | Add-Member -Type NoteProperty -Name "DistinguishedName" -Value $($obj.DistinguishedName)
            $result | Add-Member -Type NoteProperty -Name "GPODisplayName" -Value $gp.DisplayName
            $result | Add-Member -Type NoteProperty -Name "GPOID" -Value $gp.GpoId
            $result | Add-Member -Type NoteProperty -Name "Target" -Value $gp.Target
            $result | Add-Member -Type NoteProperty -Name "Enabled" -Value $gp.Enabled
            $result | Add-Member -Type NoteProperty -Name "Enforced" -Value $gp.Enforced
            $result | Add-Member -Type NoteProperty -Name "Order" -Value $gp.Order
            $a += $result
        }
        # End of Foreach loop for GPO. 
    $i++
} # End of script.
```

## **3. Output**

Once you got the result, you need to store it into a CSV file so that you can present it to the client & client IT team can run this whenever they want to know the GPO's applied on each OU or to keep it as an inventory or they can schedule this job to run weekly or monthly. 

```powershell
$filepath = "GPOresults" + (Get-date -Format "dd-MM-yyyy HH:mm:ss") + ".Csv"
# The above command creates a filepath and name to store the GP Results. 
$a | Export-Csv -Path $filepath -NoTypeInformation
```

Chiru