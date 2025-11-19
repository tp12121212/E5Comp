# Using the Graph API to assign sensitivity labels to SharePoint Online files with site administrator permissions and global administrator consent
This is an example of using the Graph API to assign sensitivity labels to SharePoint Online files. This example grants permissions to an application for a specific site only, rather than granting permissions for the entire SPO site. This is pattern No. 2 in the table below. Note that configuration No. 3 is currently not available with the pay-as-you-go API. [Reference: Known Limitations of the Metered API](https://learn.microsoft.com/ja-jp/graph/metered-api-overview#known-limitations)

| No. | Permission Type | Permission | Consent Type | Availability in Labeling API |
| --- | ---------------------- | :---: | :---: | :---: |
| 1 | Application Permission | Sites.ReadWrite.All | Administrator Consent | Available |
| 2 | Application Permission | Sites.Selected | Administrator Consent | Available |
| 3 | Delegated Permission | Sites.Selected | User Consent | Not Available |

#### Application Permission
Authenticates as an application and operates with the permissions granted directly to the application, regardless of the calling user's permissions.
#### Delegated Permission
Authenticates as each user, but the application operates on the user's behalf with a subset of the user's permissions, either through user consent or organization-wide consent from an administrator.

### To manage site permissions using Microsoft Graph Command Line Tools
For a specific SPO site, an administrator must grant the Microsoft Graph Command Line Tools (14d82eec-204b-4c2f-b7e8-296a70dab67e), the tool used to grant permissions to the application we're creating, the Sites.FullControll.All delegated permission, which allows it to manage the site on behalf of the user.

| No. | Permission Type | Permission | Consent | Availability in Labeling API |
| --- | ---------------------- | :---: | :---: | :---: |
| 4 | Delegated Permissions | Sites.FullControll.All | Administrator Consent | Available |

Assigning sensitivity labels using the Graph API incurs a pay-as-you-go fee of $0.0018 per operation, and requires an Azure subscription.

## Prerequisites
### 1. Register an Application
1. Copy the tenant ID from the Entra ID Overview page.
2. Register an application from the Entra ID App Registration page.
3. Copy the Application (Client) ID value from the overview page of the created application.
4. In the Management API Permissions section, select Add Permissions, select Microsoft Graph Application Permissions, and grant the Sites.Selected permission.
5. Grant administrator consent for the granted permissions (administrator privileges required).
6. Create a new Client Secret from the Management Certificates and Secrets section and copy the value.

### 2. Enable Pay-As-You-Go APIs
1. Access the Azure Portal with your subscription account and launch Azure Cloud Shell in a PowerShell session from the menu in the upper right corner.
2. Execute the following command:
```
$app="Application ID obtained in steps 1-2"
$rgName="LabelingByGraph"
$name="LabelingAccounts2"
az group create --name $rgName --location westus
az resource create --resource-group $rgName --name $name --resource-type Microsoft.GraphServices/accounts --properties "{""appId"": ""$app""}" --location Global
```

### 3. Prepare the scripting environment
Launch PowerShell on your device with administrative privileges and run the following once beforehand:
```PowerShell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
Install-Module -Name ExchangeOnlineManagement
Install-Module -Name Microsoft.Graph
```

### 4. Sensitivity label GUID Understanding
**This method requires administrator privileges**
Another option is to install the MPIP client and use Get-FileLabel to check labeled files.
```PowerShell
$labelName="Confidential"
Connect-IPPSSession
$labels=Get-Label|?{$_.ContentType.IndexOf("File") -ne -1}
$labels|?{$_.DisplayName -eq $labelName}|select GUID
disconnect-ExchangeOnline
```

### 5. Grant permission for the tenant administrator to manage sites using the MgGraph app.
**Administrator privileges required**
Execute the following from PowerShell.
```PowerShell
# When executing the following, sign in as a tenant administrator and check "Consent on behalf of your organization."
Connect-MgGraph -NoWelcome -Scopes "Sites.FullControl.All"
Disconnect-MgGraph
```

### 6. Grant Read & Write permissions to the labeling app via MgGraph. Grant Permissions
Execute the following from PowerShell.
```PowerShell
#Variables
$app="The application ID value obtained in steps 1-2"
$appName="Name of the labeling app"

#Specify the SPO site containing the files you want to label. Note that you must include a : after the FQDN and the site URL, omitting the https://.
$sitePath="xxx.sharepoint.com:/sites/Labeling2:"

#When executing the following, sign in as a site administrator.
Connect-MgGraph -NoWelcome -Scopes "Sites.FullControl.All"

$site=Get-MgSite -SiteId $sitePath

#Grant permissions to the labeling app
$params = @{
roles = @("write","read")
grantedToIdentities= @(
@{application = @{
id = $app
displayName =$appName
}
}
)
}
New-MgSitePermission -SiteId $site.Id -BodyParameter $params
Disconnect-MgGraph
```

## Script Sample
### Labeling a Single File Using the Graph API
```PowerShell
#Environment Variables
$tenant="Tenant ID obtained in step 1-1 above"
$app="Application ID value obtained in step 1-3 above"
$sec="Application Client Secret value obtained in step 1-6 above"
$label="Sensitivity Label GUID obtained in step 4 above"

#Specifying the SPO site containing the file you want to label. Note that you must include a : after the FQDN and the site URL, omitting the https://.
$sitePath="xxx.sharepoint.com:/sites/Labeling2:"
#Name of the document library containing the file you want to label
$libraryName="Document"
#Name of the file you want to label
$fileName="MeetingMinutes1.docx"

#Connect as a Graph Custom Application
$sec2 = ConvertTo-SecureString -String $sec -AsPlainText -Force
$sec3 = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $app, $sec2
Connect-MgGraph -NoWelcome -ClientSecretCredential $sec3 -TenantId $tenant

#Get Site
$site=Get-MgSite -SiteId $sitePath

#Get Drive
$drive=Get-MgSiteDrive -SiteId $site.id -Filter "Name eq '$libraryName'"

#Get File
$file=Get-MgDriveItemChild -DriveId $drive.Id -DriveItemId "root" -Filter "Name eq '$fileName'"

#Prepare Parameters
$params = @{
"sensitivityLabelId"=$label
"assignmentMethod"="standard"
"justificationText"="Labeled by Graph"
}

# Specify the target file by URI
$uri = ("https://graph.microsoft.com/v1.0/sites/{0}/drives/{1}/items/{2}/assignSensitivityLabel" -f $site.Id,$drive.Id,$file.Id)

# <Currently not working> Apply label (there may be a delay of a few minutes until the label is applied)(There are some restrictions.)
#Labeling cannot be performed using delegated permissions from an authenticated user. Pay-as-you-go APIs cannot be used without application-specific authentication.
Invoke-MgGraphRequest -Method "POST" -Uri $uri -Body $params
```
## Reference URL
[Practical Graph: Assign Sensitivity Labels to SharePoint Online Files](https://practical365.com/assignsensitivitylabel-api/)