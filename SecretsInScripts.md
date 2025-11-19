# Credential Management Methods for Office 365 Management PowerShell Scripts
When running various Office 365 management scripts unattended and periodically, credentials cannot be entered interactively, so they must be stored in some form in advance.
Of course, the easiest way is to embed the ID and password in the script, but if the script is leaked, an ID with administrative privileges can easily be stolen.
For this reason, this article summarizes methods for storing and retrieving credentials that can be used for such scripts.

| Method | Overview | Advantages | Notes |
| ---- | ---- | ---- | ---- |
| ① Using AppId and Certificates | Register an application for administrative connections in Azure AD, grant appropriate permissions, and link it to a separately created certificate. Various scripts connect using the application's AppId and certificate with its private key. | Credentials are not written into the script, allowing you to manage them separately. Permissions are also easily restricted because no administrator account is used. | Supported services are limited. Certificate management is required. Password management for certificates with private keys also needs to be considered. |
| ② Embed SecureString in the script | Prepare a pre-encrypted password using an encryption key specific to the user profile on the device and write it directly into the script. | Since the encrypted password is only valid on the same device and with the same user profile, it limits reusability in the event of a script leak. | Credentials cannot be managed centrally. |
| ③ Use the OS credential manager | Register credentials in the Windows OS credential manager and retrieve them from the script. | Credentials can be managed separately from scripts without being included in the script. Updates and backups can also be made from the OS UI. IDs and passwords can be managed centrally. | If the OS environment is compromised, they are easily extracted by an attacker. |
| ④ Use Azure Automation's credential management function | When using Azure Automation to periodically run PowerShell scripts, you can call and use credentials registered in your Automation account. | Credentials can be managed separately from scripts without being included in the script. Updates can also be made from the Azure UI. IDs and passwords can be managed centrally. | Can only be used when running scripts in an Azure Automation environment. If the Azure environment is compromised, they are easily extracted by an attacker. |

# 1. Using an AppId and Certificate
Register an application for management connections in Azure AD, grant appropriate permissions, and associate it with a separately created certificate. Various scripts connect using the application's AppId and certificate with its private key.
## Service Support Status
| Services | PowerShell Cmdlets | Certificate Authentication | (Reference: Authentication with Azure AD Access Token) |
| ---- | ---- | ---- | ---- |
| Exchange | Connect-ExchangeOnline | Supported in 2.0.3 (2020/09/21) | Not Supported |
| Compliance | Connect-IPPSSession | Supported in 2.0.6-Preview5 (2022/03/17) <br> (Run Install-Module -Name ExchangeOnlineManagement -AllowPrerelease) | Not Supported |
| Azure AD | Connect-AzureAD | Supported | Supported |
| SharePoint | Connect-SPOSerivce | Not Supported | Not Supported |
| Teams | Connect-MicrosoftTeams | Not Supported | Supported |

## Azure AD Registering an application on the above
Follow this guide to register the application, grant the necessary permissions, create a self-signed certificate, and bind the certificate.

## Certificate-Based Connection
````
# Password for the private key certificate .pfx (this may also be managed in the same way as steps ② and ③)
$Password = "xxxx"
$SecPass=ConvertTo-SecureString -String $Password -AsPlainText -Force

Connect-IPPSSession -AppId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" -CertificateFilePath "C:\Cert\PowerShell.pfx" -Organization "xxxx.onmicrosoft.com" -CertificatePassword $SecPass
````

# ② Embed a SecureString in a Script
This method involves preparing a pre-encrypted password using an encryption key specific to the user profile on the device, and then writing it directly into the script.
## Pre-generate credentials
````
$Password = "xxxx"
$SecPass=ConvertTo-SecureString -String $Password -AsPlainText -Force
# Copy the output of the following command as a string.
ConvertFrom-Securestring -securestring $SecPass
````
## Embedding and Using Encrypted Credentials in a Script
````
$Id="xxxx@xxxx.onmicrosoft.com"
#Embed the encrypted string from the previous step into the script
$EncPass="01000000d08c9ddf0115d1118c7a00c04fc297eb01000000513c089145a37b4894b08f5f410292f100000000020000000000106600000001000020000000daaf3caa201856f67fd45ff2fb215c6d3855620ae57924588b709cd1f0c10296 000000000e8000000002000020000000ed0981d67f1d1e2a396bb609a8d387246ca802c745 dd9950d7992d16090523d51000000058d8a486dedc1835f25a89bfbd8d110940000000e8fd0 9690f1f1a3a8d4ff6bb522a455801e5514a8944225231afaf2aeaabb545924b3aa4d2c2cda3d594ee1b800afcdd84e3ac04d3189cb3c13dc9cee8b79306"
#Generating Credentials
$SecPass= convertto-securestring -string $EncPass
$Credential= New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $Id, $SecPass

Connect-ExchangeOnline -credential $Credential
````

# ③ Using the OS Credential Manager
Register credentials in the Windows OS Credential Manager and retrieve them from a script.
## Preparation
Run the following in an administrator PowerShell session:
````
Install-Module CredentialManager
````
## Pre-registering credentials
````
$Id="xxxx@xxxx.onmicrosoft.com"
$Password = "xxxx"
New-StoredCredential -Target "Office 365" -Username $Id -Password $Password -Persist Enterprise
````
## Obtaining and using credentials in a script
````
$Credential=Get-StoredCredential -Target "Office 365"
Connect-ExchangeOnline -credential $Credential
````
## Accessing the UI
You can launch the Windows Credential Manager from the Control Panel or by pressing Windows key + R and entering the following command in the Run dialog:
````
control /name Microsoft.CredentialManager
````

# ④ Use Azure Automation's credential management feature
When using Azure Automation to periodically run PowerShell scripts, you can use credentials registered in your Automation account. Separately, configure Automation to load modules such as ExchangeOnlineManagement, and set the ID and password for the Automation account credentials, such as "Office 365."
## Obtaining and using credentials in a script
````
$Credential = Get-AutomationPSCredential -Name "Office 365"
Connect-ExchangeOnline -credential $Credential
````