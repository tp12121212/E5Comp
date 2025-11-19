# Send email notifications for Endpoint DLP operations
This is a sample solution implementation that uses a PowerShell script configured in Azure Automate to retrieve file uploads to the cloud and write operations to removable media, which can be monitored by Endpoint DLP, from the Microsoft 365 audit log.
The script then deduplicates the data and writes it to a SharePoint list. It also uses Power Automate, triggered by the addition of a new item to the SharePoint list, to send custom email notifications.
In this sample, notifications are sent for the FileCopiedToRemovableMedia and FileUploadedToCloud logs.
The structure and setup are the same as the sample [here](https://github.com/YoshihiroIchinose/E5Comp/blob/main/ExternalSharingMonitoring.md) that sends email notifications for external sharing operations, so please refer to that as well.

The general operation mechanism is as follows:
1. Prepare a SharePoint list to record Endpoint DLP operations.
2. Set up a process to send an email notification to the user who performed the operation when a new item is added to the SharePoint list using Power Automate.
3. Use Azure Automate to periodically extract specific label operations from the audit log and write the logs to the list in 1 while eliminating duplicates.

Considerations
1. Azure Automate is not expensive, but there is a pay-per-use fee based on the amount of processing.
2. Power Automate email notifications are sent by the user who configured them, and the Office 365 built-in license has a processing limit of 6,000 notifications per day.
3. Since duplicate-free label operations are added to the list, the range of logs extracted by Azure Automate can overlap. If you want timely notifications within a few hours, narrow the range of logs extracted by Azure Automate to the most recent few hours and set the schedule execution frequency to a few hours.
4. Because a Power Automate flow linked to a newly created list item is used, one operation = one email notification; email notifications summarizing multiple operations are not possible.
5. If you want to re-send email notifications for testing purposes, delete the log for the corresponding operation from the SharePoint list. This will allow Power Automate to send email notifications when the log is re-registered in Azure Automate.

# Preparation
## 1. Prepare a SharePoint site to export EndpointDLP operations
1. Create a dedicated SharePoint team site named CustomNotification.
1. Create a blank list named EndpointDLP in the site contents of the created site.
### EndpointDLP
Add the following columns, excluding the title. For ease of use in PowerShell, create the columns using alphanumeric characters as shown below.
If you mistype a column name, simply changing the column name will not correct the internal column name; delete the column and recreate it.
| Column Type | Column Name |
|---|---|
| Single Line of Text | User, Item, Operation, EnforcementMode, TargetFilePath, OriginatingDomain, TargetDomain, PolicyName, RuleName, ClientIP, DeviceName, Application, Sha1, Sha256, FileType, EvidenceFile, RemovableMediaDeviceAttributes|
| Date and Time (Include Time) *| Time |
| Yes/No | RMSEncrypted, JitTriggered, Notified|
| Number | FileSize |
| Multi-Line Text | SensitiveInfoTypeData|

*Select the Date and Time type and set Include Time to Yes. Also, in the LabelActivities list settings, set the index to Time under Indexed Columns.
When logs are written to this list, it will look like this:
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/EDLPNotif01.png"/>

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
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/EDLPNotif02.png"/>

#### Aure Automation Sample Script

```
$Credential = Get-AutomationPSCredential -Name "Office 365"
$SiteUrl="https://xxxx.sharepoint.com/sites/CustomNotification"
$LabelActivitiesList="EndpointDLP"
$HoursInterval=48
#Log acquisition scope is the past 48 hours as a test environment. Time Range
$date=Get-Date
$Start=$date.addHours($HoursInterval*-1).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
$End=$date.ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")

#Setting Values ​​to an Object Function
Function AddMember{
Param($a,$b,$c)
add-member -InputObject $a -NotePropertyName $b -NotePropertyValue $c
}

#Converting Sensitive Information to Multi-Line Text Function
Function PurseSIT{
Param($a)
$b=ConvertFrom-Json $a
$output=""
foreach($i in $b){
$detail=@()
foreach($d in $i.SensitiveInformationDetailedClassificationAttributes){
$detail+=$d.Confidence.ToString() +"x"+$d.Count
}
$line=[string]::Join(",", $detail)
$output+=$i.SensitiveInfoTypeName+":"+$i.Confidence+"x"+$i.Count+" ("+$line+")"+"`n"
}
return $output
}

#Retrieve 5,000 audit logs x up to 10 times (50,000 total) and store them in $global:output. Function
Function ExtractAuditLog{
Param($type,$op)
if($type -eq $null){return}
$itemcount=0
for($i = 0; $i -lt 10; $i++){
$result=Search-UnifiedAuditLog -RecordType $type -StartDate $global:Start -EndDate $global:End -SessionId $type -Operations $op -SessionCommand ReturnLargeSet -ResultSize 5000
"Query for $type, Round("+($i+1)+"): "+$result.Count.ToString() + " items"
$global:output += $result
$itemcount += $result.Count
if($result.count -ne 5000){break}
}
"$type Total: "+$itemcount.ToString() + " items"
}
enum EnforcementMode {
None = 0
Audit = 1
Warn = 2
WarnAndBypass = 3
Block = 4
}
#3 Extract DLP operation details from $global:output and write them to $global:csv
Function FormatLabelActitivyLog {
foreach($i in $global:output){
$AuditData=$i.AuditData|ConvertFrom-Json
$line = New-Object -TypeName PSObject

#Handling attributes with different column names or that require processing
AddMember $line "LogId" $Auditdata.Id
AddMember $line "User" $i.UserIds
AddMemberamber $line "Time" $i.CreationDate.ToString("yyyy-MM-ddTHH:mm:ssZ") 
AddMember $line "Operation" $i.Operations 
AddMember $line "Item" $AuditData.ObjectId 
AddMember $line "EnforcementMode" ([EnforcementMode].GetEnumName($Auditdata.EnforcementMode)) 
AddMember $line "SensitiveInfoTypeData" (PurseSIT (ConvertTo-Json -Compress -Depth 10 $AuditData.SensitiveInfoTypeData)) 
AddMember $line "EvidenceFile" $AuditData.EvidenceFile.FullUrl 
AddMember $line "RemovableMediaDeviceAttributes" $AuditData.RemovableMediaDeviceAttributes 
AddMember $line "PolicyName" $AuditData.PolicyMatchInfo.PolicyName
AddMember $line "RuleName" $AuditData.PolicyMatchInfo.RuleName

#Column names in the log match the destination column names, and no processing is required.
$att=@("ClientIP","DeviceName","Application","Sha1","Sha256","FileSize","RMSEncrypted","TargetFilePath",
"OriginatingDomain","TargetDomain","JitTriggered","FileType")

foreach($a in $att){
AddMember $line $a ($AuditData.$a)
}
$global:csv+=$line
}
}

#Connect to Exchange Online
Connect-ExchangeOnline -credential $credential
$csv=@()

#Get DLPEndpoint Log
$output=@()
ExtractAuditLog "DLPEndpoint" "FileCopiedToRemovableMedia,FileUploadedToCloud"
FormatLabelActitivyLog

Disconnect-ExchangeOnline -Confirm:$false

#Delete duplicates of current list items and do not upload them
Connect-PnPOnline -Url $SiteUrl -credentials $Credential
$CAML="<Query><Where><Geq><FieldRef Name='Time'/><Value Type='DateTime' IncludeTimeValue='TRUE'>$Start</Value></Geq></Where></Query>"
$csv2 = {$csv}.Invoke()
foreach($item in (Get-PnPListItem -list $LabelActivitiesList -PageSize 1000 -Query $CAML)){
$target=-1
for($i=0;$i -lt $csv2.count;$i++){
if($csv2[$i].LogId -eq $item.FieldValues["Title"]){
$target=$i
continue
}
}
if($target -ge 0){
$csv2.RemoveAt($target)
}
}
"Newly identified total EndpointDLP activities since $Start"+": "+$csv2.count

#Add search results not in the list as new list items
#Used CSOM because batch processing of Add-PnPListItem did not allow writing timestamps in UTC.
$ctx=get-pnpcontext
$list = $ctx.get_web().get_lists().getByTitle($LabelActivitiesList)
$count=0
foreach($item in $csv2){
$lic = New-Object Microsoft.SharePoint.Client.ListItemCreationInformation
$i = $list.AddItem($lic)
$i.set_item("Title", $item.LogId)

#Write as is if the column names match.
$att=@("User", "Time", "Operation", "Item", "EnforcementMode", "PolicyName", "RuleName", "ClientIP",
"DeviceName", "Application", "FileSize", "SensitiveInfoTypeData", "RMSEncrypted", "TargetFilePath",
"RemovableMediaDeviceAttributes", "OriginatingDomain", "TargetDomain", "JitTriggered",
"FileType", "EvidenceFile")

foreach($a in $att){
$i.set_item($a,$item.$a)
}

#Column names containing numbers are converted to special column names in SharePoint, so write them according to the internal column names.
$i.set_item("_x0053_ha1", $item.Sha1)
$i.set_item("_x0053_ha256", $item.Sha256)

$i.update()
$count++
#If there are many writes, first reflect them in 100 items.
if($count % 100 -eq 0){
$ctx.ExecuteQuery()}
}
$ctx.ExecuteQuery()
"Endpoint DLP activities were synchronized with the list."
Disconnect-PnPOnline
```
## 3. Setting up email notifications using Power Automate
1. From the EndpointDLP list's Integrations menu, select Power Automate and select Create Flow.
1. Select the "Send a customized email when a new SharePoint list item is added" flow and create it.
1. Delete the "Get My Profile (V2)" and "Send Email" steps from Edit.
1. Add a "Condition" to the "Control" section of the new step.
1. Specify the dynamic content "User" as the condition value, and set the conditions to "Contains" and "\".
(This applies to local processes unrelated to Azure AD.) (For users, this will exclude them from the notification.)
1. Add the "Get Manager (V2)" action to the "If No" section.

(For internal user operations, include the internal user who shared the file in the "To" field and their manager in the "CC" field to send an email notification.)
1. Add the list column [User] in the "User (UPN)" field using dynamic content.
1. Next, add the "Send Email (V2)" action for "Outlook."
1. Specify the list column [User] in the "To" field using dynamic content.
1. Enter "[EDLP Notification] File Export Operation" in the subject line.
1. Click "Show advanced options" and add the manager [Mail] in the "CC" field using dynamic content.
1. Include approximately the following content in the body of the email ([ ] refers to a list column in the dynamic content).
```
The file has been exported.
Please confirm that this is a necessary business operation.
User: [User]
Time (UTC): [Time]
Exported file: [Item]
Operation: [Operation]
Operation availability: [EnforcementMode]
DLP match
DLP policy: [PolicyName]
DLP rule: [RuleName]
Detected sensitive information:
[SensitiveInfoTypeData]

For USB export
External media information:
[RemovableMediaDeviceAttributes]

For cloud upload
Destination: [TargetDomain]
```
1. Add a new step after the step and add an "Update Item" action.
1. Specify the "CustomNotification" site for the site address and "EndpointDLP" for the list name.
1. Specify the ID as the list column ID in the dynamic content.
1. Specify the title as the list column Title in the dynamic content.
1. Leave the values ​​for "RMSEncrypted", "JitTriggered", and "Notified" blank.
1. Set "Notified" to Specify "Yes"
1. Save the flow

### The entire Power Automate flow after configuration
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/EDLPNotif03.png"/>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/EDLPNotif04.png"/>

### Example of an email notification sent
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/EDLPNotif05.png"/>
In addition, in Power Automate, you can create a shared mailbox and grant "send on behalf" permissions to send email notifications from the shared mailbox account.