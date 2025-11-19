# List guest users configured on each SharePoint Online and OneDrive for Business site (SPO Shell)
This script uses the SharePoint Online Management Shell from a script created in Azure Automation to retrieve a list of guest users on each SharePoint Online and OneDrive for Business site.
The retrieved external user list information is stored as a CSV file in a document library for a specific SharePoint Online site.
Note that due to the query mechanism, guest users can be detected if an external user is registered in a SharePoint group, including via an external invitation in Teams, or if permissions are directly granted to the external user. However, indirect permissions granted to external users via security groups cannot be retrieved.
You can check the groups an external user belongs to by checking the external user's memberof attribute in Azure AD.

## Preparation
1. Create an Azure Automation account in your Azure environment.
2. In your Azure Automation account, add the following module from "Modules" -> "Browse Gallery."
(Runtime version is 5.1)
SharePointOnline.CSOM
Microsoft.Online.SharePoint.PowerShell
3. In the Azure Automation account, go to "Credentials" -> "Add Credentials" and register the ID and password of an account with SharePoint Online administrator privileges and permission to post to the specified SharePoint Online site under the name "Office 365."
5. Create a PowerShell runbook with runtime version 5.1 by selecting "Runbooks" -> "Create a Runbook" in your Azure Automation account.
6. Copy and paste the following script into the runbook.
7. Save and publish the SharePoint site URL and the relative URL (without the FQDN) of the file destination, as appropriate.
8. Start the runbook and verify its operation.

## Output when Azure Automation is executed
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/SPOExt1.png">
## Output CSV file
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/SPOExt2.png">

## Script body
```
$siteUrl="https://xxxx.sharepoint.com/sites/CustomNotification/"
$targeturl ="/sites/CustomNotification/Shared Documents/ExternalUsersFromSPO.csv"
$adminSiteUrl="https://xxxx-admin.sharepoint.com"
$OutputFile=[System.Environment]::GetFolderPath("Desktop")+"\ExternalUsersFromSPO.csv"
$Credential = Get-AutomationPSCredential -Name "Office 365"

Function AddMember{ 
Param($a,$b,$c) 
add-member -InputObject $a -NotePropertyName $b -NotePropertyValue $c
}

Connect-SPOService -Url $adminSiteUrl -Credential $Credential

$Sites=Get-SPOSite -Limit ALL -IncludePersonalSite $true
$externalUsers=@()
Foreach($s in $sites){ $users=Get-SPOExternalUser -Position 0 -PageSize 50 -SiteUrl $s.Url 
"$($users.count) external users in $($s.Url)" 
If($users.count -eq 50){"Obtained external users might be capped."} 
Foreach($u in $users){ 
$line = New-Object -TypeName PSObject 
AddMember $line "Site" $s.Url 
AddMember $line "DisplayName" $u.DisplayName 
AddMember $line "UniqueId" $u.UniqueId 
AddMember $line "Email" $u.Email 
AddMember $line "LoginName" $u.LoginName 
$externalUsers+=$line 
}
}

"Total external users found:"+$userList.count
$externalUsers|Export-Csv -Path $OutputFile -NoTypeInformation -Encoding UTF8

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