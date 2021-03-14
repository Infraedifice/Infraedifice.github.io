---
title: "App Proxy"
excerpt: "Extract list of Azure AD App proxy applications with PowerShell"
date:   "2021-03-15"
categories: 
- "App proxy", "PowerShell"
tags: "App proxy", "PoweShell"
---
Recently during the Azure AD Application Proxy (App Proxy) deployment project with one of our clients, I was asked to give a list of applications that are on-boarded into App Proxy. Namely the client needed the Internal Url, External URL & DisplayName of the application. Usually there are two ways of doing that. One way is to export them manually into a csv file or alternatively: automate it! If there are small number of applications, then exporting them manually wouldn’t take that long but a long-term solution is to automate it.

This is where I would like to harness the Power of PowerShell. Unfortunately, there is no single PowerShell cmdlet which will give us list of App Proxy applications. So, after doing a bit of research online and testing PowerShell cmdlets I came to this final version of script which uses Azure AD PowerShell connection to retrieve the results.

The PowerShell Script here is categorized into 3 parts.

1. **Function (To check the connection to Azure PowerShell)**
2. **Core Script**
3. **Output**

## 1.**Function (To check the connection to Azure PowerShell)**
This part will check if the user is connected to Azure PowerShell
By using this we can avoid multiple connections to Azure PowerShell and Azure AD MFA (If Enabled) requests

``` PowerShell
### Start of Function to check the Logins###

Function Login-check
{
    try
     {
        $Az = Get-AzureADDomain -ErrorAction Stop
     }
    catch
        {
            $Fault = $PSItem.message.exception
        }
    if ($Az)
        {
           Write-Host -ForegroundColor Cyan "User logged into Azure"
        }
    else
        {
           Write-Host -ForegroundColor red "Login Required"
           Connect-AzureAD
        }
}

```
## 2.**Core Script**

**This part of the script will retrieve us the information about all the applications in Azure AD with ObjectId**
**I initiated a counter so that we can see which application the script is processing**
**I loop each of these applications and check if they’re onboarded to app-proxy**
**For the ones that are not on-boarded to app-proxy the PowerShell cmdlet gives an error which is captured**
**I defined an empty variable in the loop to capture all the required information**

``` Powershell
### Start of core Script###
Login-check
$App = Get-AzureADApplication -All $true | Select-Object Displayname, ObjectId
$i = 1
$Appproxy_List = @()
$count = $app.count
Foreach ($apps in $app)
{
    Write-Progress -Activity "Getting $($apps.DisplayName) App-proxy information" -Status "$i of $count" -PercentComplete ($i/$count*100)
      try {
            $proxy = Get-AzureADApplicationProxyApplication -ObjectId "$($apps.Objectid)" -ErrorAction Stop
            #####The Above cmdlet is used to get list of App-proxy Applications###########
            $result = " " | Select-Object DisplayName, ObjectId, InternalUrl, ExternalUrl
            $result.DisplayName = $($apps.Displayname)
            $result.ObjectId = $($apps.Objectid)
            $result.InternalUrl = $($proxy.InternalUrl)
            $result.ExternalUrl = $($proxy.ExternalUrl)
            $Appproxy_List += $result
          } #End of Try Statement
      catch {   
             $Problem = $PSItem.exception.message         
            } #End of Catch statement
    $i++
} #End of Foreach loop
### End of Core ###

```

## 3.**Output**

**I defined a saving directory where the output is saved.**
**I defined file name with date and time.**
**Script will create a directory if it doesn’t exist else it will save the file in the directory.**
**Lastly, we will export the output as CSV file to the path specified.**

```PowerShell
### Output ###
$SavingDirectory = "C:\Temp\"
If ((Test-Path -Path $SavingDirectory) -eq $false) {New-Item -ItemType Directory -Path $SavingDirectory | Out-Null}
$Filepath = $SavingDirectory + "App-proxy Application list Master Copy" + " " + (Get-Date -Format "dd-MM-yyyy_HH.mm.ss") + ".csv"
$Appproxy_List | Export-Csv -Path $Filepath -NoTypeInformation
##### End of Script##############################################

```
Hope this post is helpful in finding Azure App proxy application using PowerShell