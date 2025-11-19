The English version is [here](https://github.com/YoshihiroIchinose/E5Comp/blob/main/ExternalSharingMonitoring_en.md)
# Send email notifications for unauthorized B2B external sharing operations
As a complement to DLP, we'll introduce a sample procedure for capturing file sharing operations with external users from unauthorized domains via SharePoint Online, OneDriver for Business, Teams, etc., and sending email notifications based on those operations from the audit log.
The general mechanism for this is as follows:
1. Prepare a SharePoint list to manage allowed domains.
2. Prepare a SharePoint list to record sharing operations.
3. When a new item is added to the SharePoint list using Power Automate, set up a process to send an email notification to the user who performed the sharing operation.
4. Use Azure Automate to periodically extract sharing operations to external users in unauthorized domains from the audit log, eliminating duplicates and writing the log of new sharing operations to the list in step 2.

Considerations
1. Azure Automate is not expensive, but there is a pay-per-use fee based on the amount of processing.
2. Power Automate email notifications are sent by the user who configured them, and the built-in Office 365 license has a processing limit of 6,000 notifications per day.
3. Since sharing operations are added to the list after deduplicating them, the range of logs extracted by Azure Automate can overlap. If you want timely notifications within a few hours, narrow the range of logs extracted by Azure Automate to the most recent few hours and set the schedule execution frequency to a few hours.
4. If you want to send email notifications again for testing purposes, delete the log for the relevant sharing operation from the SharePoint list. This will allow Power Automate to send email notifications when the log is re-registered in Azure Automate.

# Preparation
## 1. Prepare the SharePoint site for exporting external sharing operations
1. Create a dedicated SharePoint team site named CustomNotification.
1. Create two blank lists with the following names from the site contents of the created site.
### AllowedDomains
This is a list with titles only, with no additional columns. Add domains to exclude from notifications. A backward match with the guest user ID is performed to exclude allowed users by domain. The following screen shows the list after adding allowed domains.
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/Notification6.png"/>

### SharingActivities
Add the following columns in addition to the title: To make it easier to use in PowerShell, create columns using alphanumeric characters as follows.
| Column Name | Title | User | Guest | Time | Operation | SharedItem | AdditionalData | Notified |
|-------|----|----|----|----|----|----|----|----|
| Column Type | Existing Single Line of Text | Single Line of Text | Single Line of Text | Date and Time* | Single Line of Text | Single Line of Text | Multiple Lines of Text | Yes/No (Default: No) |
| Information to Store | GUID to Identify Log | User Who Performed the Operation | Invited Guest | Time of Invite | Type of Operation | Shared Item | Store Other Additional Information | Flag to Indicates Whether Notification Processing Was Performed |

*Select the Date and Time type and set Include Time to Yes. Also, in the SharingActivities list settings, set the index to Time under Indexed Columns.
When logs are written to this list, it will look like this.
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/Notification1.png"/>

## 2. Setting Up Your Azure Environment
1. Create an Azure Automation Account
1. In your Azure Automation account, add the following three modules from "Modules" -> "Browse Gallery."
(Runtime version is 5.1)
SharePointOnline.CSOM
PnP.PowerShell
ExchangeOnlineManagement
1. In your Azure Automation account, go to "Credentials" -> "Add Credentials" and register the ID and password of an account with permissions to extract audit logs and post to the specified SharePoint Online site under the name "Office 365."
1. In your Azure Automation account, go to "Runbooks" -> "Create a Runbook" and create a PowerShell runbook with runtime version 5.1.
1. Copy and paste the following script into the runbook you created.
1. Modify the SharePoint site URL and list name in the script as appropriate, save it, and publish it.
1. "Start" the runbook you created and check its operation.
1. Set a schedule such as Daily if necessary.
When you run this script in Azure Automation, the number of processed logs will be output, as shown below.
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/Notification2B.png"/>

#### Aure Automation Sample Script
```
#Variables
$Credential = Get-AutomationPSCredential -Name "Office 365"
$SiteUrl="https://xxxx.sharepoint.com/sites/CustomNotification/"
$AllowedDomainList="AllowedDomains"
$SharingActivitiesList="SharingActivities"
$HoursInterval=48

#Log acquisition scope is the past 48 hours as a test environment. Time Range
$date=Get-Date
$Start=$date.addHours($HoursInterval*-1).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
$End=$date.ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")

#Get Domain Allow List
Connect-PnPOnline -Url $SiteUrl -credentials $Credential
$AllowedDmains=@()
foreach($item in Get-PnPListItem -list $AllowedDomainList -PageSize 1000){
$AllowedDmains+=$item.FieldValues["Title"].ToLower()
}

#Set Value to Object Function
Function AddMember{
Param($a,$b,$c)
add-member -InputObject $a -NotePropertyName $b -NotePropertyValue $c
}

#Function for making Azure AD B2B guest IDs easier to read
Function ExtractGuest{
Param($a)
$a=$a.replace("#EXT#","#ext#")
If($a.Contains("#ext#")){
return $a.Substring(0,$a.IndexOf("#ext#")).replace("_","@")
}
return $a
}

#Function for determining whether a domain is included in the allowed list
Function IsAllowed{
Param($a)
$a=$a.ToLower()
foreach($i in $AllowedDmains){
if($a.EndsWith($i)){return $true}
}
return $false
}

#5,000 audit logs x up to 10 times = 50,000 Function that retrieves the data and stores it in $global:output
Function ExtractAuditLog{ 
Param($type,$op) 
if($type -eq $null){return} 
$itemcount=0 
for($i = 0; $i -lt 10; $i++){ 
$result=Search-UnifiedAuditLog -RecordType $type -StartDate $global:Start -EndDate $global:End -SessionId ($type+$op) -Operations $op -SessionCommand ReturnLargeSet -ResultSize 5000 
"Query for $type, $op, Round("+($i+1)+"): "+$result.Count.ToString() + " items" 
$global:output+=$result 
$itemcount+=$result.Count 
if($result.count -ne 5000){break} 
} 
"$type, $op Total: "+$itemcount.ToString() + " items"
}

#1. Obtaining a log of the operation of adding a guest user to a SharePoint group and granting permissions
Connect-ExchangeOnline -credential $Credential
$output=@()
ExtractAuditLog "SharePointSharingOperation" "AddedToGroup"

#If the result is not null, first count one result. If multiple results are returned, obtain their count.
$count=0
if($output -ne $null){$count=1}
if($output.count -ne $null){$count=$output.count}
"Site sharing activities logs since $Start"+": "+$count

$csv=@()
foreach($i in $output){
$AuditData=$i.AuditData|ConvertFrom-Json
#Excludes permissions set by the SPO app account when creating the site
if($i.UserIds -eq "app@sharepoint"){continue}
#Excludes adding non-guests or guests from allowed domains
If($AuditData.TargetUserOrGroupType -ne "Guest" ){continue}
$guest=ExtractGuest $AuditData.TargetUserOrGroupName
If(isAllowed($guest)){continue}
$line = New-Object -TypeName PSObject
AddMember $line "User" $i.UserIds
AddMember $line "Guest" $guest
AddMember $line "Time" $i.CreationDate.ToString("yyyy-MM-ddTHH:mm:ssZ") 
AddMember $line "Operation" "Site Shared" 
AddMember $line "SharedItem" $AuditData.SiteUrl 
AddMember $line "LogId" $AuditData.CorrelationId 
AddMember $line "AdditionalData" $AuditData.EventData 
$csv+=$line
}
"Unallowed site sharing activities since $Start"+": "+$csv.count

#Merge logs with the same CorrelationID
$GroupedCsv=@()
foreach($i in ($csv|Group-Object LogId)){ 
$line = $i.Group[0] 
$AdditionalData=@() 
foreach($d in $i.Group){ 
$AdditionalData+=$d.AdditionalData 
} $line.AdditionalData=$AdditionalData -join "`r`n"
$GroupedCsv+=$line
}
"Unallowed site sharing activities merged since $Start"+": "+$GroupedCsv.count

#2. Obtain logs for operations where files are shared directly with guest users
$output=@()
ExtractAuditLog "SharePointSharingOperation" "AddedToSecureLink"

#If the result is not null, first count one result. If multiple results are returned, obtain their count.
$count=0
if($output -ne $null){$count=1}
if($output.count -ne $null){$count=$output.count}
"File sharing activities logs since $Start"+": "+$count

$count=0
foreach($i in $output){
$AuditData=$i.AuditData|ConvertFrom-Json
#Excludes sharing to non-guests and guests from allowed domains.
If($AuditData.TargetUserOrGroupType -ne "Guest" ){continue}
$guest=ExtractGuest $AuditData.TargetUserOrGroupName
If(isAllowed($guest)){continue}
$line = New-Object -TypeName PSObject
AddMember $line "User" $i.UserIds
AddMember $line "Guest" $guest
AddMember $line "Time" $i.CreationDate.ToString("yyyy-MM-ddTHH:mm:ssZ")
AddMember $line "Operation" "File Shared"
AddMember $line "SharedItem" $AuditData.ObjectId
AddMember $line "LogId" $AuditData.CorrelationId
AddMember $line "AdditionalData" $AuditData.EventData
$GroupedCsv+=$line
$count++
}
"Unallowed file sharing activities since $Start"+": "+$count

Merge logs with the same CorrelationID in #1 and #2, prioritizing File Shared.
$GroupedCsv2=@()
foreach($i in ($GroupedCsv|Group-Object LogId)){
$line = $i.Group[0]
$AdditionalData=@()
foreach($j in $i.Group){
$AdditionalData+=$j.AdditionalData
if($j.Operation -eq "File Shared"){$line=$j}
}
$line.AdditionalData=$AdditionalData -join "`r`n"
$GroupedCsv2+=$line
}
"Unallowed total sharing activities since $Start"+": "+$GroupedCsv2.count

#3. Obtaining logs for adding guest users to an existing group
$output=@()
ExtractAuditLog "AzureActiveDirectory" "Add member to group."
#If the result is not null, first count one result. If multiple results are returned, obtain their count.
$count=0
if($output -ne $null){$count=1}
if($output.count -ne $null){$count=$output.count}
"Adding member activities since $Start"+": "+$count
Disconnect-ExchangeOnline -Confirm:$false

$count=0
foreach($i in $output){
$AuditData=$i.AuditData|ConvertFrom-Json
#Excludes additions other than guests.
If(!$AuditData.ObjectId.Contains("#EXT#")){continue}
$guest=ExtractGuest $AuditData.ObjectId 
If(isAllowed($guest)){continue} 
$line = New-Object -TypeName PSObject 
AddMember $line "User" $i.UserIds 
AddMember $line "Guest" $guest 
AddMember $line "Time" $i.CreationDate.ToString("yyyy-MM-ddTHH:mm:ssZ") 
AddMember $line "Operation" "Guest Added to Group" 
AddMember $line "SharedItem" $AuditData.ModifiedProperties[1].NewValue 
AddMember $line "LogId" $AuditData.ID 
AddMember $line "AdditionalData" $AuditData.ModifiedProperties[0].NewValue 
$count++ 
$GroupedCsv2+=$line
}
"Unallowed adding member activities since $Start"+": "+$count

#Remove duplicates of current list items and do not upload them
$CAML="<Query><Where><Geq><FieldRef Name='Time'/><Value Type='DateTime' IncludeTimeValue='TRUE'>$Start</Value></Geq></Where></Query>"
$GroupedCsv2 = {$GroupedCsv2}.Invoke()
foreach($item in (Get-PnPListItem -list $SharingActivitiesList -PageSize 1000 -Query $CAML)){
$target=-1
for($i=0;$i -lt $GroupedCsv2.count;$i++){
if($GroupedCsv2[$i].LogId -eq $item.FieldValues["Title"]){
$target=$i
continue
}
}
if($target -ge 0){
$GroupedCsv2.RemoveAt($target)
}
}
"Newly identified total sharing activities since $Start"+": "+$GroupedCsv2.count

#Add search results not in the list as new list items
#Used CSOM because batch processing of Add-PnPListItem did not allow writing timestamps in UTC.
$ctx=get-pnpcontext
$list = $ctx.get_web().get_lists().getByTitle($SharingActivitiesList)
$count=0
foreach($item in $GroupedCsv2){
$lic = New-Object Microsoft.SharePoint.Client.ListItemCreationInformation
$i = $list.AddItem($lic)
$i.set_item("Title", $item.LogId)
$i.set_item("User", $item.User)
$i.set_item("Guest", $item.Guest)
$i.set_item("Time", $item.Time)
$i.set_item("Operation", $item.Operation)
$i.set_item("SharedItem", $item.SharedItem)
$i.set_item("AdditionalData", $item.AdditionalData)
$i.update()
$count++
#If there are many writes, reflect them per 100 items
if($count % 100 -eq 0){
$ctx.ExecuteQuery()}
}
$ctx.ExecuteQuery()
"File sharing activities were synchronized with the list."
Disconnect-PnPOnline
```

## 3. Setting up email notifications using Power Automate
1. From the SharingActivities list menu, select Power Automate from the Integrations menu and select Create Flow.
1. Select the "Send a customized email when a new SharePoint list item is added" flow and create it.
1. Delete the "Get My Profile (V2)" and "Send Email" steps from the edit menu.
1. Add a "Condition" to the "Control" section of the new step.
1. Specify the dynamic content "User" as the condition value, and set the conditions "Contains" and "#ext#".
(If the sharing operation is performed by an external user, the email notification will not be sent to the user, but will be sent to a designated administrator.)
1. Add an "Outlook" "Send Email (V2)" action to the "If Yes" step.
1. Set the administrator's email address as the recipient.
1. Enter "Guest user sharing to unauthorized domain" in the subject line.
1. Include approximately the following content in the body of the message ([ ] refers to a list column in the dynamic content):
A guest user shared to an unauthorized domain.
Please confirm the content.
User: [User]
Time (UTC): [Time]
Shared Item: [SharedItem]
To: [Guest]

1. Add the "Get Manager (V2)" action to the "If No" section.

(If sharing is with an internal user, send an email notification with the internal user who shared in the "To" field and their manager in the "CC" field.)
1. Add the list column [User] in the "User (UPN)" dynamic content.
1. Next, add the "Send Email (V2)" action for "Outlook."
1. Specify the list column [User] in the "To" dynamic content.
1. Enter "Sharing to Unauthorized Domain" in the subject line.
1. Click "Show advanced options" and add the manager [Mail] in the "CC" dynamic content.
1. Include approximately the following content in the body of the email ([ ] refers to a list column in the dynamic content):
Sharing to an unauthorized domain has occurred. If this was unintentional, please unshare the email.
If this is a business-related operation, please contact the Help Desk and request domain permission.
User: [User]
Time (UTC): [Time]
Shared Item: [SharedItem]
Shared To: [Guest]

1. Add a new step after this and add an "Update Item" action.
1. Specify the "CustomNotification" site in the site address field and "SharingActivities" in the list name field.
1. Specify the ID as the list column's [ID] using dynamic content.
1. Specify the title as the list column's [Title] using dynamic content.
1. Set "Notified" to "Yes."
1. Save the flow.

### Full Power Automate flow after configuration.
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/Notification7.png"/>

### Example of an email notification sent.
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/Notification3.png"/>
In addition, Power Automate allows you to create a shared mailbox and grant "send on behalf" permissions to send email notifications from the shared mailbox account.