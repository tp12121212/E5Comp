## May become unavailable in the future
This script uses legacy tokens that allow access to all the functionality of the previous MDA. The recommended OAuth 2.0 tokens will have stricter authorization scope, and the APIs used in this script will no longer work. Therefore, this script may no longer work in the future. Reference: [MDA Token](https://learn.microsoft.com/en-us/defender-cloud-apps/api-authentication)

## About Throttling
MDA has a limit of 30 calls per minute. Depending on the MDA environment, API calls that exceed this limit can behave in two ways: asynchronously, where an immediate throttled error is returned, or synchronously, where a one-minute wait for a response is made and the correct result is returned. This script was tested in a synchronous environment, so error handling in asynchronous environments has not been properly verified.

# List guest users set for each SharePoint Online and OneDrive for Business site (MDA)
This script calls the Defender for Cloud Apps REST API from a script created in Azure Automation to retrieve files shared externally in SharePoint Online and OneDrive for Business, and then retrieves a list of guest users for each site based on their permission settings. The retrieved external user list is stored as a CSV file in a document library for a specific SharePoint Online site. Due to the query mechanism, it can detect guest users registered in SharePoint groups, including those invited externally in Teams, and external users who have been granted permissions directly. However, it does not detect indirect permissions granted to external users via security groups. You can check the groups that external users belong to by viewing the external user's memberof attribute in Azure AD.

## Preparation
1. Create an Azure Automation account in your Azure environment.
2. In your Azure Automation account, add the following modules from "Modules" -> "Browse Gallery."
(Runtime version is 5.1)
SharePointOnline.CSOM
3. In your Azure Automation account, go to "Credentials" -> "Add Credentials" and register the ID and password of an account with post permissions to the specified SharePoint Online site under the name "Office 365."
4. Access the Defender for Cloud Apps [API Token](https://security.microsoft.com/cloudapps/settings?tabid=apiTokens) page and obtain a 64-character alphanumeric token in advance. Save the 64-character token as a "variable" in your Azure Automation account under the name "MDAToken." Enable encryption for this "variable."
5. Create a PowerShell runbook with runtime version 5.1 by selecting "Runbooks" -> "Create a Runbook" in your Azure Automation account.
6. Copy and paste the following script into the runbook.
7. Edit the Defender for Cloud Apps URL, SharePoint site URL, relative URL of the file destination (excluding FQDN), and list of internal domains in the script as appropriate, save, and publish.
8. Start the runbook and verify its operation.

## Output when Azure Automation is executed
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/MDAExt1.png">

## Output CSV file
The Item records the file URL, folder URL, SharePoint group name, etc., depending on the shared target.
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/MDAExt2.png">

## Script body
````
#MDA URL
$Base="https://xxxx.portal.cloudappsecurity.com/"
#Target CSV location in SPO
$siteUrl="https://xxxx.sharepoint.com/sites/CustomNotification/"
$targeturl="/sites/CustomNotification/Shared Documents/ExternalUsers.csv"
#Internal domain list
$InternalDomains=@("xxxx.onmicrosoft.com")
#Maximum files to retrieve
$ResultSetSize=3000

#fixed parameters (No need to modify)
$Credential = Get-AutomationPSCredential -Name "Office 365"
$Token=Get-AutomationVariable -Name "MDAToken"
#a filter condition for getting externally shared files from SPO and OD4B
$filter='{"service":{"eq":[15600,20892]},"sharing":{"eq":[2,3,4]}}'
#This filter will return files directly shared with external users with specific domains but won't include files shared via site permissions.
#$filter='{"service":{"eq":[15600,20892]},"collaborators.withDomain":{"eq":["hotmail.com","gmail.com","icloud.com"]}}'
#Temporal output location
$OutputFile=[System.Environment]::GetFolderPath("Desktop")+"\ExternalUsers.csv"
#Maximum files to retrieve at a sigle query
$batchSize=100

Function AddMember{ Param($a,$b,$c) 
add-member -InputObject $a -NotePropertyName $b -NotePropertyValue $c
}

Function IsExternalDomains{ 
Param($a) 
$a=$a.ToLower() 
foreach($i in $InternalDomains){ 
if($a.EndsWith($i)){return $false} 
} 
return $true
}

$loopcount = [int][Math]::Ceiling($ResultSetSize / $batchSize)
$headers=@{"Authorization" = "Token "+$Token}
$output=@()
"Start getting externally shared files from MDA"
For($i=0;$i -lt $loopcount; $i++){ 
$limit=$batchSize 
if($loopcount -1 -eq $i){$limit=$ResultSetSize % $batchSize} if($limit -eq 0){$limit=$batchSize} 
$Body=@{ 
"skip"=0 + $i*$batchSize 
"limit"=$limit 
"filters"=$filter 
"sortField"="modifiedDate" 
"sortDirection"="desc" 
} 
$Uri=($base+"/api/v1/files/").Replace("//api/","/api/")
do { 
$retryCall = $false 
"Loop: $i, From " +$i*$batchSize 
try { 
$res=Invoke-RestMethod -Uri $Uri -Method "Post" -Headers $headers -Body $Body 
} 
catch { 
if ($_ -like '504' -or $_ -like '502' -or $_ -like '429') { 
$retryCall = $true Start-Sleep-Seconds 5 
} 
ElseIf ($_ -match 'throttled') { 
$retryCall = $true 
Start-Sleep -Seconds 60 
} 
else { 
throw $_ 
} 
} 
} 
while ($retryCall) 
$output+=$res.data 
if($res.data.Count -lt $batchsize){break}
}

$userlist=@()
$GroupsWithExternalUsers=@{}
foreach($row in $output){ 
foreach($c in $row.collaborators){ 
If($c.type -eq 2 -and $c.accessLevel -eq 2){#Group with external users 
$group = New-Object -TypeName PSObject 
AddMember $group "Item" $c.name 
AddMember $group "Id" $c.id If($GroupsWithExternalUsers[$row.sitePath] -eq $null){$GroupsWithExternalUsers[$row.sitePath]=@()} 
$found=$false 
foreach($g in $GroupsWithExternalUsers[$row.sitePath]){ 
If($g.Id -eq $group.id){$found=$true} 
} 
If(!$found){$GroupsWithExternalUsers[$row.sitePath]+=$group} 
} 
If($c.type -eq 1 -and $c.accessLevel -eq 2){#direct assignments of external users 
If($c.name -eq "NT Service\SPTimerV4"){continue} 
$line = New-Object -TypeName PSObject 
AddMember $line "Site" $row.siteCollection 
AddMember $line "Item" $row.filePath 
AddMember $line "User" $c.name 
AddMember $line "Email" $c.email $userList+=$line 
} 
}
}
"Files with direct assignments of external users:"+$userList.count
"Sites with SPO Groups including external users:"+$GroupsWithExternalUsers.count

foreach($Site in $GroupsWithExternalUsers.Keys){ 
foreach($g in $GroupsWithExternalUsers[$Site]){ 
$groupId="$Site|$($g.Id)" 
$groupId= [System.Web.HttpUtility]::UrlEncode($groupId) 
$appId=20892 
If($Site.StartsWith("/personal/")){$appId=15600} 
"Getting external users from $($g.Item) in $Site" $Uri=($base+"/api/v1/get_group/?appId=$appId&groupId=$groupId&limit=100").Replace("//api/","/api/") 
do { 
$retryCall = $false 
try { 
$res=Invoke-RestMethod -Uri $Uri -Method "GET" -Headers $headers 
} 
catch{ 
if ($_ -like '504' -or $_ -like '502' -or $_ -like '429') { 
$retryCall = $true 
Start-Sleep-Seconds 5 
} 
ElseIf ($_ -match 'throttled') { 
$retryCall = $true 
Start-Sleep -Seconds 60 
} 
else { 
throw $_ 
} 
} 
} 
while ($retryCall) 
Foreach($u in $res.group.membersList){ 
$line = New-Object -TypeName PSObject 
if($u.emailAddress -ne $null -and (IsExternalDomains($u.emailAddress))){ 
AddMember $line "Site" $Site 
AddMember $line "Item" $g.Item 
AddMember $line "User" $u.Name 
AddMember $line "Email" $u.emailAddress 
$userList+=$line 
} 
} 
}
}
"Total external users found:"+$userList.count
$userList|Export-Csv -Path $OutputFile -NoTypeInformation -Encoding UTF8

#Uploading a csv file to SPO
Load-SPOnlineCSOMAssemblies
$ctx = New-Object Microsoft.SharePoint.Client.ClientContext($siteUrl)
$username =$Credential.UserName
$password = $Credential.Password
$cre=$null
$count=0
while($cre -eq $null -and $count -lt 10){ 
$count++ 
try{$cre = New-Object Microsoft.SharePoint.Client.SharePointOnlineCredentials($username,$password)} 
catch{ 
"Authentication Error!" 
$_.Exception.Message 
Start-Sleep -s 5 
}
}
$ctx.Credentials=$cre

$fs = new-object System.IO.FileStream($OutputFile ,[System.IO.FileMode]::Open,[System.IO.FileAccess]::Read)
[Microsoft.SharePoint.Client.File]::SaveBinaryDirect($ctx,$targeturl , $fs, $true)
$fs.Close()
$ctx.Dispose()
"Upload completed."
````