# Checking external sharing logs, primarily for B2B collaboration
The DLP features of Office 365's SharePoint Online and OneDrive for Business can block or warn users when confidential information is shared externally.
However, while DLP can identify which files meet DLP policies, it does not specifically track which domains external users shared with, with the exception of email. Therefore, even if external sharing is permitted, you may still need to know who shared with whom from the logs.
This script extracts external sharing logs in CSV format, including guest users, for the following three operations: 1. Adding a guest user to a SharePoint group and granting permissions on a SharePoint Online / OneDriver for Business site
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/Log1.png">
2. Sharing a file directly to a guest user on SharePoint Online / OneDriver for Business
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/Log2.png">
3. Adding a guest user to an existing group
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/Log3.png">

However, the above logs only cover operations that require Azure AD B2B Federation.
They do not cover operations such as Teams Connect, which uses B2B Direct, or anonymous links.

The script on this page retrieves up to 5,000 logs for each operation, but for retrieving more than 5,000 logs, please also refer to [here](https://github.com/YoshihiroIchinose/E5Comp/blob/main/Office365Audit.md). For information on checking UI-based audit logs for other SharePoint sharing operations, please refer to [here](https://learn.microsoft.com/ja-jp/microsoft-365/compliance/audit-log-sharing?view=o365-worldwide).
## Script
```
#ID/Password to use for connection
$Id="xxx@xxx.onmicrosoft.com"
$Password = "xxxxx"

#Variables
$Startdate="2023/01/01"
$Enddate="2023/03/08"
$OutputFolder=[System.Environment]::GetFolderPath("Desktop")+"\"

#Credential Generation
$SecPass=ConvertTo-SecureString -String $Password -AsPlainText -Force
$Credential= New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $Id, $SecPass

Import-Module ExchangeOnlineManagement
Connect-ExchangeOnline -credential $Credential

#1. Adding a Guest User to SharePoint Log of operations for adding users to a group and granting permissions
$RecordType="SharePointSharingOperation"
$Operation="AddedToGroup"
$output=Search-UnifiedAuditLog -RecordType $RecordType -StartDate $Startdate -EndDate $Enddate -Operations $Operation -ResultSize 5000 -SessionCommand ReturnNextPreviewPage|?{$_.UserIds -ne "app@sharepoint"}

$csv=@()
foreach($row in $output){
$JsonRaw=$row.AuditData|ConvertFrom-Json
#Excludes non-guest additions
If($JsonRaw.TargetUserOrGroupType -ne "Guest" ){continue}
$data=$row |select UserIds,Operations,CreationDate
add-member -InputObject $data -NotePropertyName CorrelationId -NotePropertyValue $JsonRaw.CorrelationId
add-member -InputObject $data -NotePropertyName SiteUrl -NotePropertyValue $JsonRaw.SiteUrl
add-member -InputObject $data -NotePropertyName EventData -NotePropertyValue $JsonRaw.EventData
$GuestId=$JsonRaw.TargetUserOrGroupName
If($GuestId.Contains("#ext#")) 
{$GuestId=$GuestId.Substring(0,$GuestId.IndexOf("#ext#")).replace("_","@")}
add-member -InputObject $data -NotePropertyName TargetUserOrGroupName -NotePropertyValue $GuestId
$csv+=$data
}

#Output as CSV file
$filePath=$OutputFolder+"SPO_AddedToGroup"+(Get-Date -Format "yyyyMMdd-HHmm")+".csv"
$csv|Export-csv -Path $filePath -Encoding UTF8 -NoTypeInformation

#2. Log of File Sharing Operations Directly with Guest Users
$RecordType="SharePointSharingOperation"
$Operation="AddedToSecureLink"
$output=Search-UnifiedAuditLog -RecordType $RecordType -StartDate $Startdate -EndDate $Enddate -Operations $Operation -ResultSize 5000 -SessionCommand ReturnNextPreviewPage

$csv=@()
foreach($row in $output){
$JsonRaw=$row.AuditData|ConvertFrom-Json
#Exclude sharing with non-guests
If($JsonRaw.TargetUserOrGroupType -ne "Guest" ){continue}
$data=$row |select UserIds,Operations,CreationDate
add-member -InputObject $data -NotePropertyName CorrelationId -NotePropertyValue $JsonRaw.CorrelationId
add-member -InputObject $data -NotePropertyName FileUrl -NotePropertyValue $JsonRaw.ObjectId
add-member -InputObject $data -NotePropertyName EventData -NotePropertyValue $JsonRaw.EventData
$GuestId=$JsonRaw.TargetUserOrGroupName
If($GuestId.Contains("#ext#")) 
{$GuestId=$GuestId.Substring(0,$GuestId.IndexOf("#ext#")).replace("_","@")}
add-member -InputObject $data -NotePropertyName TargetUserOrGroupName -NotePropertyValue $GuestId
$csv+=$data
}

#CSV Output as a file
$filePath=$OutputFolder+"SPO_AddedToSecureLink"+(Get-Date -Format "yyyyMMdd-HHmm")+".csv"
$csv|Export-csv -Path $filePath -Encoding UTF8 -NoTypeInformation

#3. Log of guest user addition to an existing group
$RecordType="AzureActiveDirectory"
$Operation="Add member to group."
$output=Search-UnifiedAuditLog -RecordType $RecordType -StartDate $Startdate -EndDate $Enddate -Operations $Operation

$csv=@()
foreach($row in $output){
$JsonRaw=$row.AuditData|ConvertFrom-Json
#Excludes non-guest additions
If(!$JsonRaw.ObjectId.Contains("#EXT#")){continue}
$data=$row |select UserIds,Operations,CreationDate
add-member -InputObject $data -NotePropertyName LogID -NotePropertyValue $JsonRaw.ID
add-member -InputObject $data -NotePropertyName GroupName -NotePropertyValue $JsonRaw.ModifiedProperties[1].NewValue
add-member -InputObject $data -NotePropertyName GroupID -NotePropertyValue $JsonRaw.ModifiedProperties[0].NewValue
$AddedGuest=$JsonRaw.ObjectId.Substring(0,$JsonRaw.ObjectId.IndexOf("#EXT#")).replace("_","@")
add-member -InputObject $data -NotePropertyNameAddedGuest -NotePropertyValue $AddedGuest
$csv+=$data
}
$csv|Export-csv -Path ($OutputFolder+"AAD_AddedToGroup"+(Get-Date -Format "yyyyMMdd-HHmm")+".csv") -Encoding UTF8 -NoTypeInformation

Disconnect-ExchangeOnline -Confirm:$false
````