---
title: "Orchestration of Exchange online mailbox size reports using Azure automation & Microsoft Graph API"
excerpt: "Automate M365 reports using Azure runbook & Graph API"
header: 
   og_image: /assets/images/teaser-logo.png
   teaser: /assets/images/teaser-logo.png
date:   "2021-02-21"
categories: 
- "Automation"
tags: 
- "Azure Runbook"
- "Azure automation"
- "Microsoft Graph API"
- "Orchestration"
---

![Solution Overview](/assets/images/Email orchestration/Solution-Overview.png)

Reporting in M365 has come a long way. We all know that anyone with the proper access can view the M365 reports in the admin portal with a splendid graphical view. What if the operations team wants to get the notifications to a mailbox if the mailbox size reaches a specific limit (example use case)? My initial thoughts were to use SendGrid to send emails, but I thought we could use graph API to send emails rather than relying on a third-party email solution. I planned to use Azure automation to query the mailbox size from the Exchange online & use Microsoft Graph API to send emails. 

We need the following pre-requisites to achieve the above. 

## **Pre-requisites**
- **Azure Automation Account**
- **Exchange online module**
- **Microsoft Graph API**
- **Azure Key Vault**

## Azure Automation Account ##

You need to have an Azure automation account to run Azure automation jobs. Upon automation account & new runbook creation, you need to assign the Exchange administrator permission to run as account service principal. You can give the permissions via PowerShell or from GUI. 
You need to install the Exchange online module PowerShell in the automation account. You can get that from the modules gallery, as shown below. 

![Assigning Exchange administrator role to service principal](/assets/images/Email orchestration/Azure-Roles&Responsibilities.png)
Exchange Administrator role in Azure AD roles & responsibilites.

![Assigning Exchange administrator role to service principal](/assets/images/Email orchestration/ExchangeAdmnistrator-Runasaccount-Serviceprincipal.png)
Assigning Exchange administrator role to run as account service principal. 

![Install Exchange online module](/assets/images/Email orchestration/Exchange Online Module - Automation account.png)
Exchange online powershell module

![Install Exchange online module](/assets/images/Email orchestration/Exchange Online-Browse Gallery - Microsoft Azure.png)
Exchange online powershell module in the gallery. 


## App registration in Azure AD ##

Create and register a new application in Azure AD to send emails using Microsoft Graph API. Upon creating and registering the application, you need to assign Microsoft Graph API permissions to send emails on behalf of users. You need to choose application permissions in the Microsoft Graph API.

![Microsoft Graph API Permissions](/assets/images/Email orchestration/Request API permissions - Microsoft Azure.png)
Microsoft Graph API permissions.


![Microsoft Graph API Permissions](/assets/images/Email orchestration/Microsoft Graph API permissions - Microsoft Azure.png)
Assigning Microsoft Graph API application permissions. 

![Microsoft Graph API Permissions](/assets/images/Email orchestration/Microsoft Graph API -Permissions.png)
Microsoft Graph API permissions on the Azure AD Application.


Now, we need to authenticate the application from the custom code in the runbook. I will do this by configuring a client secret in the application. Make sure to copy the secret upon creation as you can't see the client secret again. 

![Azure AD Application Client Secret](/assets/images/Email orchestration/Azure AD Application- Client Secret.png)
Client secret in Azure AD Application. 


## Azure key Vault ##

We will leverage Azure Key Vault to securely store the client secret we created for the Azure AD Application. We manually generate a new secret in the Azure key vault. Upon holding the client secret, we will assign "Get" permissions to the "run as account" in the key vault access policies to retrieve the secret. We don't need any other permissions as we are just retrieving the secret. 

![Azure Key Vault Secret](/assets/images/Email orchestration/Azure-key-Vault  - Microsoft Azure.png)
Generate secret in Azure key Vault.

![Azure Key Vault Secret](/assets/images/Email orchestration/Create a secret - Azure Key Vault - Microsoft Azure.png)


Run the below code to send secure emails from the Azure automation runbook using Graph API. 

I categorized the code into three parts. 

- **Import modules & Azure Automation account connection**
- **Mailbox size report from Exchange online**
- **Microsoft Graph API to send emails**

## Import modules & Azure Automation account connection ##

You need to import the Azure, Azure key vault and Exchange online modules and connect to the Azure automation account from the runbook as shown below. 

```powershell
Import-module AZ.Accounts
Import-module Az.Keyvault
Import-module Exchangeonlinemanagement

### Connecting to Azure Automation Account ###
try {

    $connection = Get-AutomationConnection -Name AzureRunAsConnection
    $tenantName = "test.onmicrosoft.com"
    $Azureaccount = Connect-AzAccount `
        -ServicePrincipal `
        -TenantId $Connection.TenantId `
        -ApplicationId $Connection.ApplicationId `
        -CertificateThumbprint $Connection.CertificateThumbprint 
}
catch
{
    if (!$Connection)

    {
        ## Run as Account not configured ##
        $error = "Connecting to $connection failed"
        throw $error
    }
    else
    {
        ## Something else went wrong ##
        write-error -message $_.Exception.message
        throw $_.Exception
    }
}

```


## Mailbox size report from Exchange online ##

Authentication to Exchange online will be done using the "run as account".   Run the below PowerShell code to extract the mailbox size report for mailboxes greater than a specific limit.

```powershell

### Connecting to Exchange online ###
Connect-ExchangeOnline -CertificateThumbprint $connection.CertificateThumbprint -AppId $connection.ApplicationID -Organization $tenantName -Verbose
# Date Format
$CurrentDate = (get-date).ToString('yyyy-MM-dd')
$CurrentDate
#Mailboxes
$Mailboxes = Get-Mailbox -ResultSize Unlimited | Select-Object Name, Alias, ExchangeGuid
$Output = @()
$i = 1
Foreach ($Mailbox in $Mailboxes)
{
    $Mailboxsize = Get-MailboxStatistics -Identity "$($mailbox.ExchangeGuid)" | Select-Object DisplayName, MailboxTypeDetail, @{name=”TotalItemSize”; expression={[math]::Round(($_.TotalItemSize.ToString().Split(“(“)[1].Split(” “)[0].Replace(“,”,””)/1GB),2)}}, IsArchiveMailbox

    if ($mailboxsize.TotalItemSize -gt "90") {
        
        $Mailboxesabovelimit = New-Object -TypeName psobject -Property @{

            DisplayName = $Mailboxsize.DisplayName
            Emailaddress = $mailbox.PrimarySmtpAddress
            MailboxType = $Mailboxsize.MailboxTypeDetail
            MailboxSize = $Mailboxsize.TotalItemSize
            ArchiveStatus = $Mailboxsize.IsArchiveMailbox
            
        }
        $Output += $Mailboxesabovelimit
    }
    $i++
}

$Output

$filename = "MailboxSizes"+"$currentdate"+"."+"csv"

$filename

$path = "$Env:temp\$filename"

$path

$Output | Export-Csv -Path $path -NoTypeInformation

```


## Microsoft Graph API to send emails ##

Run the below code to send the emails using Graph API securely. You can modify the email template as per your requirement. I used a shared mailbox to send notification emails.

```powershell

if ($Output)
{

$base64string = [convert]::ToBase64String((Get-Content -path $path -Encoding byte))

#### key vault Name ###

$keyvault = "Enter your Keyvault name"

### Secret Name in the key Vault ###

$keyvaultSecretName = "Enter your secret name from the Keyvault"

### Application ID of the Azure AD App registration ###

$Application_ID = "xxxx-xxxx-xxxxx-12345"

### Tenant ID of the Azure AD App registration ###

$TenantID = $connection.TenantId

### From Address of the Email from Exchange online ###

$from = "notification@test.com"

### To Address of the email ###

$to = "testuser@test.com"

### Getting the secret from Azure Keyvault ###

$secret = Get-AzKeyVaultSecret -VaultName $KeyVault -Name $KeyVaultSecretName


$Securestringpointer = [System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($secret.SecretValue)


try {

   $secretValueText = [System.Runtime.InteropServices.Marshal]::PtrToStringBSTR($Securestringpointer)

} 

finally {
  
   [System.Runtime.InteropServices.Marshal]::ZeroFreeBSTR($Securestringpointer)

}


$client_secret = $secretValueText

$request = @{
        Method = 'POST'
        URI    = "https://login.microsoftonline.com/xxxxxxxxxx/oauth2/v2.0/token"
        body   = @{
            grant_type    = "client_credentials"
            scope         = "https://graph.microsoft.com/.default"
            client_id     = $Application_ID
            client_secret = $client_secret
        }
    }
$token = (Invoke-RestMethod @request).access_token

#$token

## Build the Microsoft Graph API request

## Header ##

$headers = @{
    "Authorization" = "Bearer $($token)"
    "Content-type"  = "application/json"
}

## Sending email ##

$sendURL = "https://graph.microsoft.com/v1.0/users/$from/sendMail"

$mailbody = @"
                    {
                        "message": {
                          "subject": "MailboxSize Report - $CurrentDate",
                          "body": {
                            "contentType": "HTML",
                            "content": "Email content."
                          },
                          
                          "toRecipients": [
                            {
                              "emailAddress": {
                                "address": "$to"
                              }
                            }
                          ]
                          ,"attachments": [
                            {
                              "@odata.type": "#microsoft.graph.fileAttachment",
                              "name": "$FileName",
                              "contentType": "text/csv",
                              "contentBytes": "$base64string"
                            }
                          ]
                        },
                        "saveToSentItems": "false"
                      }
"@

Invoke-RestMethod -Method POST -Uri $sendURL -Headers $headers -Body $mailbody

}

else
{
    Write-Verbose -Verbose "None of the mailbox size is greater than 90GB"
}

```

You can schedule the runbook as per your requirement. 

I hope you enjoy the blog! 

Chiru