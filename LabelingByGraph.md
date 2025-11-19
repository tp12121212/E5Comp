# Assigning sensitivity labels to SharePoint Online files using the Graph API
This is a sample that uses the Graph API to assign sensitivity labels to SharePoint Online files.
Assigning sensitivity labels using the Graph API incurs a pay-as-you-go fee of $0.00185 per operation, an Azure subscription is required.

## Prerequisites
### 1. Register an Application
1. Copy the tenant ID from the Entra ID Overview page.
2. Register an application from the Entra ID App Registration page.
3. Copy the Application (Client) ID value from the overview page of the created application.
4. Grant Sites.ReadWrite.All permission to the application in the Management API Permissions section.
5. Grant administrator consent for the granted permissions.
6. Create a new client secret from the Management Certificates and Secrets section and copy the value.

Add SPO to the application. If you want to grant permissions only for a specific site, rather than the entire site, please see [here](https://github.com/YoshihiroIchinose/E5Comp/blob/main/LabelingByGraph2.md).

### 2. Enabling the Pay-As-You-Go API
1. Access the Azure Portal and launch Azure Cloud Shell in a PowerShell session from the menu in the upper right corner.
2. Execute the following command:
```
$app="Application ID obtained in steps 1-2 above"
$rgName="LabelingByGraph"
$name="LabelingAccounts"
az group create --name $rgName --location westus
az resource create --resource-group $rgName --name $name --resource-type Microsoft.GraphServices/accounts --properties "{""appId"": ""$app""}" --location Global
```

### 3. Preparing the Scripting Environment
Launch PowerShell with administrative privileges and run the following once beforehand:
```
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
Install-Module -Name ExchangeOnlineManagement
Install-Module -Name Microsoft.Graph
```

### 4. Finding the Sensitivity Label GUID
```
$labelName="Confidential"
Connect-IPPSSession
$labels=Get-Label|?{$_.ContentType.IndexOf("File") -ne -1}
$labels|?{$_.DisplayName -eq $labelName}|select GUID
disconnect-ExchangeOnline
```

## Script Sample
### Labeling a Single File Using the Graph API
```PowerShell
#Environment Variables
$tenant="Tenant ID obtained in step 1-1 above"
$app="Application ID value obtained in step 1-2 above"
$sec="Application Client Secret value obtained in step 1-6 above"
$label="Sensitivity Label GUID obtained in step 4 above"

#Specifying the SPO site where the file you want to label is located. Add : after the FQDN and after the site URL, without the https:// prefix. Note the inclusion of "$sitePath="xxx.sharepoint.com:/sites/label:""
#Name of the document library containing the file you want to label
$libraryName="Documents"
#Name of the file you want to label
$fileName="MeetingMinutes1.docx"

#Connect to Graph with a client secret
$sec2 = ConvertTo-SecureString -String $sec -AsPlainText -Force
$sec3 = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $app, $sec2
Connect-MgGraph -NoWelcome -ClientSecretCredential $sec3 -TenantId $tenant

#Get Site
$site=Get-MgSite -SiteId $sitePath

#Get Drive
$drive=Get-MgSiteDrive -SiteId $site.id -Filter "Name eq '$libraryName'"

#Get File
$file=Get-MgDriveItemChild -DriveId $drive.Id -DriveItemId "root" -Filter "Name eq '$fileName'"

#Prepare parameters
$params = @{
"sensitivityLabelId"=$label
"assignmentMethod"="standard"
"justificationText"="Labeled by Graph"
}

#Specify target file by URI
$uri = ("https://graph.microsoft.com/v1.0/sites/{0}/drives/{1}/items/{2}/assignSensitivityLabel" -f $site.Id,$drive.Id,$file.Id)

#Apply label (there will be a lag of a few minutes until the label is applied)
Invoke-MgGraphRequest -Method "POST" -Uri $uri -Body $params
```
### Graph API Label multiple files directly under a specific folder using this command.
Note that this will replace existing labels regardless of priority.
```PowerShell
#Environment Variables
$tenant="Tenant ID obtained in step 1-1 above"
$app="Application ID value obtained in step 1-2 above"
$sec="Application Client Secret value obtained in step 1-6 above"
$label="Sensitivity Label GUID obtained in step 4 above"

#Specify the SPO site containing the files you want to label. Note that you must include a : after the FQDN and the site URL, omitting the https://.
$sitePath="xxx.sharepoint.com:/sites/label:"
#Name of the document library containing the files you want to label
$libraryName="Documents"
#Path to the folder within the library you want to label
$folder="Confidential"
#For hierarchical labels
#$folder="Confidential/Encrypted"

#Connect to Graph with a client secret
$sec2 = ConvertTo-SecureString -String $sec -AsPlainText -Force
$sec3 = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $app, $sec2
Connect-MgGraph -NoWelcome -ClientSecretCredential $sec3 -TenantId $tenant

#Get Site
$site=Get-MgSite -SiteId $sitePath

#Get Drive
$drive=Get-MgSiteDrive -SiteId $site.id -Filter "Name eq '$libraryName'"

#Get Files in a Specific Folder
$files=Get-MgDriveItemChild -DriveId $drive.Id -DriveItemId ("root:/"+$folder+":")

#Prepare Parameters
$params = @{
"sensitivityLabelId"=$label
"assignmentMethod"="standard"
"justificationText"="Labeled by Graph"
}

#Limit to supported files
$supported=@("docx","pptx","xlsx","pdf")

Foreach($file in $files){
If(!$supported.contains($file.Name.split(".")[-1])){
continue
}
$base ="https://graph.microsoft.com/v1.0/sites/$($site.Id)/drives/$($drive.Id)/items/"
$uri=$base+$file.Id+"/extractSensitivityLabels"
$l=Invoke-MgGraphRequest -Method "POST" -Uri $uri
If($l.labels.sensitivityLabelId -eq $label){
"Skip labeling for '"+$file.Name +"' because a label has already been assigned."
}
else {
"Label '"+$file.Name +"'"
$uri=$base+$file.Id+"/assignSensitivityLabel"
Invoke-MgGraphRequest -Method "POST" -Uri $uri -Body $params
}
}
```
### Label multiple files directly under a specific folder in parallel using the Graph API.
Note that existing labels will be replaced regardless of priority.
```PowerShell
#Environment variables
$tenant="Tenant ID obtained in step 1-1 above"
$app="Application ID value obtained in step 1-2 above"
$sec="Application client secret value obtained in step 1-6 above"
$label="Sensitivity label GUID obtained in step 4 above"

#Specify the SPO site where the files you want to label are located. (Without https://)Note that you must include a : after the FQDN and the site URL.
$sitePath="xxx.sharepoint.com:/sites/label:"
#Name of the document library containing the file you want to label
$libraryName="Documents"
#Path of the folder within the library you want to label
$folder="Test Folder"

#Header for token acquisition
$body = @{
client_Id = $app
client_secret = $sec
scope = 'https://graph.microsoft.com/.default'
grant_type = 'client_credentials'
}

#Token acquisition
$uri="https://login.microsoftonline.com/$tenant/oauth2/V2.0/token"
$tokenRequest = Invoke-WebRequest -Method Post -Uri $uri -Body $body
$token = ($tokenRequest.Content | ConvertFrom-Json).access_token

#Connect to Graph with a token
$sec2 = ConvertTo-SecureString -String $token -AsPlainText -Force
Connect-MgGraph -AccessToken $sec2 -NoWelcome

#Get Site
$site=Get-MgSite -SiteId $sitePath

#Get Drive
$drive=Get-MgSiteDrive -SiteId $site.id -Filter "Name eq '$libraryName'"

#Get Files in a Specific Folder
$files=Get-MgDriveItemChild -DriveId $drive.Id -DriveItemId ("root:/"+$folder+":")

#Labeling Settings
$params = @{
"sensitivityLabelId"=$label
"assignmentMethod"="standard"
"justificationText"="Labeled by Graph"
}

#Define a Workflow for Parallel Processing. Reuse an Existing AccessToken.
Workflow Label-Files(){
param($files,$base,$sec2,$label,$params)
Foreach -parallel ($file in $files){
Connect-MgGraph -AccessToken $sec2 -NoWelcome
$uri=$base+$file.Id+"/extractSensitivityLabels"
$l=Invoke-MgGraphRequest -Method "POST" -Uri $uri
If($l.labels.sensitivityLabelId -eq $label){
"Skip labeling for '"+$file.Name +"' because it's already labeled."
}
else {
"Label '"+$file.Name +"'"
$uri=$base+$file.Id+"/assignSensitivityLabel"
Invoke-MgGraphRequest -Method "POST" -Uri $uri -Body $params
}
}
}

#Limit to supported extensions
$supported=@("docx","pptx","xlsx","pdf")
$files=$files|?{$supported.contains($_.Name.split(".")[-1])}

#Perform labeling in parallel
$base ="https://graph.microsoft.com/v1.0/sites/$($site.Id)/drives/$($drive.Id)/items/"
Label-Files $files $base $sec2 $label $params
```
### Recursively detects multiple files under a specific folder using the Graph API and labels them in parallel.
Note that existing labels will be replaced regardless of priority.
```PowerShell
#Environment variables
$tenant="Tenant ID obtained in step 1-1 above"
$app="Application ID value obtained in step 1-2 above"
$sec="Application ID value obtained in step 1-6 above" "The value of the application's client secret obtained in step 4 above."
$label="GUID of the sensitivity label obtained in step 4 above"

#Specifying the SPO site containing the files you want to label. Note that you must include a : after the FQDN and after the site URL, omitting the "https://" prefix.
$sitePath="xxx.sharepoint.com:/sites/label:"
#Name of the document library containing the files you want to label.
$libraryName="Documents"
#Path to the folder within the library you want to label.
$folder="Test Folder"

#Header for token acquisition
$body = @{
client_Id = $app
client_secret = $sec
scope = 'https://graph.microsoft.com/.default'
grant_type = 'client_credentials'
}

#Token acquisition
$uri="https://login.microsoftonline.com/$tenant/oauth2/V2.0/token"
$tokenRequest = Invoke-WebRequest -Method Post -Uri $uri -Body $body
$token = ($tokenRequest.Content | ConvertFrom-Json).access_token

#Connect to Graph with token
$sec2 = ConvertTo-SecureString -String $token -AsPlainText -Force
Connect-MgGraph -AccessToken $sec2 -NoWelcome

#Get Site
$site=Get-MgSite -SiteId $sitePath

#Get Drive
$drive=Get-MgSiteDrive -SiteId $site.id -Filter "Name eq '$libraryName'"

#Limit to supported extensions
$supported=@("docx","pptx","xlsx","pdf")

#Associative array of target files
$files=@{}

#Function to recursively dig folders
Function Get-FolderItems (){
param($folder)
$current="root:/"+$folder+":"
"Folder reference:" + $current
#Get items in a specified folder
$items=Get-MgDriveItemChild -DriveId $drive.Id -DriveItemId ($current)
Foreach($item in $items){
#If the item is a file to be labeled
If($item.Folder.ChildCount -eq $null -and $supported.contains($item.Name.split(".")[-1])){
$files[$item.Id]=$item.Name
}
#If the item is a folder
ElseIf($item.Folder.ChildCount -ge 1){
Get-FolderItems ($folder+"/"+$item.Name)
}
}
}

#Recursively detect files in a specified folder
Get-FolderItems $folder

#Labeling settings
$params = @{
"sensitivityLabelId"=$label
"assignmentMethod"="standard"
"justificationText"="Labeled by Graph"
}

#Define a workflow for parallel processing. Reuse an existing AccessToken.
Workflow Label-Files(){
param($files,$base,$sec2,$label,$params)
Foreach -parallel ($key in $files.keys){
Connect-MgGraph -AccessToken $sec2 -NoWelcome
$uri=$base+$key+"/extractSensitivityLabels"
$l=Invoke-MgGraphRequest -Method "POST" -Uri $uri
If($l.labels.sensitivityLabelId -eq $label){
"Skip labeling for '"+$files[$key]+"' because it's already labeled."
}
else {
"Label '"+$files[$key]+"'"
$uri=$base+$key+"/assignSensitivityLabel"
Invoke-MgGraphRequest -Method "POST" -Uri $uri -Body $params
}
}
}

#Perform labeling in parallel
$base ="https://graph.microsoft.com/v1.0/sites/$($site.Id)/drives/$($drive.Id)/items/"
Label-Files $files $base $sec2 $label $params
```

## Reference URL
[Practical Graph: Assign Sensitivity Labels to SharePoint Online Files](https://practical365.com/assignsensitivitylabel-api/)