# Use Azure Automation to output specific operations from Office 365 audit logs in CSV format and upload them to a SharePoint Online site.
In this sample, up to 50,000 logs from the DLPEndpoint log type ("FileAccessedByUnallowedApp," "FileCopiedToRemovableMedia," "FilePrinted," and "FileUploadedToCloud") are extracted for a 31-day period (from yesterday through 31 days ago) and uploaded to a SharePoint Online site in CSV format.
Note that only one level of logs included in the audit data is parsed and created as CSV columns.
This script also outputs CSV files with fixed names to SharePoint Online, which are updated daily. This allows you to import the CSV files from SharePoint Online as a data source into Power BI to create a daily-updated dashboard. Using this script as a base, you can also output logs of sensitivity label assignment, modification, and deletion operations in CSV format and upload them to a specific SharePoint Online site by setting $RecordType="SensitivityLabelAction" and $Operations="".

## Preparation
1. Create an Azure Automation account in your Azure environment.
2. In your Azure Automation account, add the following three modules from "Modules" -> "Browse Gallery".
(Runtime version is 5.1)
SharePointOnline.CSOM
Microsoft.Online.SharePoint.PowerShell
ExchangeOnlineManagement
3. In your Azure Automation account, go to "Credentials" -> "Add Credentials" and register the ID and password of an account with permissions to extract audit logs and post to the specified SharePoint Online site under "Office 365".
4. In your Azure Automation account, go to "Runbooks" -> "Create a Runbook" and create a PowerShell runbook with runtime version 5.1.
5. Copy and paste the following script into the runbook.
6. Edit the SharePoint site URL and the relative URL (excluding the FQDN) of the file destination in the script as appropriate, save, and publish.
7. Start the runbook and verify its operation.
8. Schedule execution as needed, such as daily.

## Uploading audit logs to SPO by specifying RecordType and Operations: Azure Automation PowerShell Script
```
#Variables
$date=Get-Date
$Startdate=$date.addDays(-31).ToString("yyyy/MM/dd")
$Enddate=$date.ToString("yyyy/MM/dd")
$RecordType="DLPEndpoint"
$outfile="C:\Report\"+$RecordType+".csv"
$siteUrl="https://xxx.sharepoint.com/sites/DLPLogs/"
$targeturl ="/sites/DLPLogs/Shared Documents/"+$RecordType+".csv"

#Operations to retrieve (can be left unspecified)
$Operations="FileAccessedByUnallowedApp","FileCopiedToRemovableMedia","FilePrinted","FileUploadedToCloud"

#Other Operations
#"ArchiveCreated","FileCopiedToClipboard","FileCopiedToRemoteDesktopSession","FileCreated"
#"FileCreatedOnNetworkShare","FileCreatedOnRemovableMedia","FileDeleted","FileDownloadedFromBrowser"
#"FileModified","FileRead","FileRenamed","RemovableMediaMount","RemovableMediaUnmount"

#Generate Credential and Connect
$Credential = Get-AutomationPSCredential -Name "Office 365"
Connect-ExchangeOnline -credential $Credential

#Generate a unique session ID string with date and time
$sessionId=$RecordType+(Get-Date -Format "yyyyMMdd-HHmm")

#Get logs in a loop of up to 5,000 x 10 times
$output=@();
for($i = 0; $i -lt 10; $i++){ 
if($Operations -ne $null -and $Operations.Length -ne 0){ 
$result=Search-UnifiedAuditLog -RecordType $RecordType -Operations $Operations -Startdate $Startdate -Enddate $Enddate -SessionId $sessionId -SessionCommand ReturnLargeSet -ResultSize 5000 
} 
else { 
$result=Search-UnifiedAuditLog -RecordType $RecordType -Startdate $Startdate -Enddate $Enddate -SessionId $sessionId -SessionCommand ReturnLargeSet -ResultSize 5000 
} 
$output+=$result 
"Query "+($i+1)+" round: "+$result.Count.ToString() + " results" 
if($result.count -ne 5000){break}
}
Disconnect-ExchangeOnline -Confirm:$false
if($output.count -eq 0){
"No data"
exit
}
"Total: "+$output.Count.ToString() + " results"

#Get fields contained in JSON from the first item for each Operation type
$OperationTypes=$output|Group-Object Operations
$FieldName=@()
foreach($Operation in $OperationTypes){
$JsonRaw=$Operation.Group[0].AuditData|ConvertFrom-Json
$FieldsInJson=$JsonRaw|get-member -type NoteProperty
foreach($f in $FieldsInJson){
if($FieldName.Contains($f.Name) -eq $false) { $FieldName+=$f.Name}
}
}

#Generate a ScriptBlock to parse JSON for use with Select-Object
$Fields="ResultIndex", "CreationDate","UserIds","Operations","RecordType"
foreach($f in $FieldName){
$sb1=[scriptblock]::Create('$JsonRaw.'+$f)
$sb2=[scriptblock]::Create('$att=$JsonRaw.'+$f+';if($att.GetType().Name -eq "Object[]" -or $att.GetType().Name -eq "PSCustomObject"){ConvertTo-Json -Compress -Depth 10 $att} else {$att}')
if($f -ne "RecordType") {$Fields+=@{Name=$f;Expression=$sb2}}
else {$Fields+=@{Name="RecordType2";Expression=$sb1}}
}

#Parse JSON and convert to CSV format
$csv=@();
foreach($row in $output){
$JsonRaw=$row.AuditData|ConvertFrom-Json
$data=$row|Select-Object -Property $Fields
$csv+=$data
}

#Output
$csv|Export-Csv -Path $outfile -NoTypeInformation -Encoding UTF8

#Loading the CSOM Assembly and Connecting to SPO
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
Start-Sleep -s 5}
}
$ctx.Credentials=$cre

$fs = new-object System.IO.FileStream($outfile ,[System.IO.FileMode]::Open,[System.IO.FileAccess]::Read)
[Microsoft.SharePoint.Client.File]::SaveBinaryDirect($ctx,$targeturl , $fs, $true)
$fs.Close()
$ctx.Dispose()
"Upload completed."
```
## Power BI Report Sample
CSV files on a SharePoint Online site can be imported into a report by selecting "Add Dataset" and then "Get File" from any workspace in the web version of Power BI. Data updated every hour is automatically retrieved.
![Sample](img/PowerBI_EDLP.png)

## Azure Automation PowerShell for uploading all types of audit logs to SPO as of July 13, 2022 Script (for verification)
```
#Variables
$global:date=Get-Date
$global:Startdate=$date.addDays(-31).ToString("yyyy/MM/dd")
$global:Enddate=$date.ToString("yyyy/MM/dd")

#Target Log
$RecordTypes="ExchangeAdmin", "ExchangeItem", "ExchangeItemGroup", "SharePoint", "SyntheticProbe", "SharePointFileOperation"
$RecordTypes+="OneDrive", "AzureActiveDirectory", "AzureActiveDirectoryAccountLogon", "DataCenterSecurityCmdlet", "ComplianceDLPSharePoint"
$RecordTypes+="Sway", "ComplianceDLPExchange", "SharePointSharingOperation", "AzureActiveDirectoryStsLogon", "SkypeForBusinessPSTNUsage"
$RecordTypes+="SkypeForBusinessUsersBlocked", "SecurityComplianceCenterEOPCmdlet","ExchangeAggregatedOperation","PowerBIAudit"
$RecordTypes+="CRM","Yammer","SkypeForBusinessCmdlets","Discovery","MicrosoftTeams","ThreatIntelligence","MailSubmission"
$RecordTypes+="MicrosoftFlow","AeD","MicrosoftStream","ComplianceDLPSharePointClassification","ThreatFinder","Project"
$RecordTypes+="SharePointListOperation","SharePointCommentOperation","DataGovernance","Kaizala","SecurityComplianceAlerts"
$RecordTypes+="ThreatIntelligenceUrl","SecurityComplianceInsights","MIPLabel","WorkplaceAnalytics","PowerAppsApp","PowerAppsPlan"
$RecordTypes+= "ThreatIntelligenceAtpContent","LabelContentExplorer","TeamsHealthcare","ExchangeItemAggregated","HygieneEvent"
$RecordTypes+="DataInsightsRestApiAudit","InformationBarrierPolicyApplication","SharePointListItemOperation","SharePointContentTypeOperation"
$RecordTypes+="SharePointFieldOperation","MicrosoftTeamsAdmin","HRSignal","MicrosoftTeamsDevice","MicrosoftTeamsAnalytics"
$RecordTypes+="InformationWorkerProtection","Campaign","DLPEndpoint","AirInvestigation","Quarantine","MicrosoftForms","ApplicationAudit"
$RecordTypes+="ComplianceSupervisionExchange","CustomerKeyServiceEncryption ","OfficeNative","MipAutoLabelSharePointItem","MipAutoLabelSharePointPolicyLocation"
$RecordTypes+="MicrosoftTeamsShifts","SecureScore","MipAutoLabelExchangeItem","CortanaBriefing","Search","WDATPAlerts","PowerPlatformAdminDlp"
$RecordTypes+="PowerPlatformAdminEnvironment","MDATPAudit","SensitivityLabelPolicyMatch","SensitivityLabelAction","SensitivityLabeledFileAction"
$RecordTypes+="AttackSim","AirManualInvestigation","SecurityComplianceRBAC","UserTraining","AirAdminActionInvestigation","MSTIC","PhysicalBadgingSignal"
$RecordTypes+="TeamsEasyApprovals","AipDiscover","AipSensitivit yLabelAction","AipProtectionAction","AipFileDeleted","AipHeartBeat","MCASAlerts"
$RecordTypes+="OnPremisesFileShareScannerDlp","OnPremisesSharePointScannerDlp","ExchangeSearch","SharePointSearch","PrivacyDataMinimization"
$RecordTypes+="LabelAnalyticsAggregate","MyAnalyticsSettings","SecurityComplianceUserChange","ComplianceDLPExchangeClassification","ComplianceDLPEndpoint"
$RecordTypes+="MipExactDataMatch","MSDEResponseActions","MSDEGeneralSettings","MSDEIndicatorsSettings","MS365DCustomDetection","MSDERolesSettings"
$RecordTypes+="MAPGlerts","MAPGPolicy","MAPGRemediation","Priva cyRemediationAction","PrivacyDigestEmail","MipAutoLabelSimulationProgress"
$RecordTypes+="MipAutoLabelSimulationCompletion","MipAutoLabelProgressFeedback","DlpSensitiveInformationType","MipAutoLabelSimulationStatistics"
$RecordTypes+="LargeContentMetadata","Microsoft365Group","CDPMlInferencingResult","FilteringMailMetadata","CDPClassificationMailItem"
$RecordTypes+="CDPClassificationDocument","OfficeScriptsRunAction","FilteringPostMailDeliveryAction","CDPUnifiedFeedback","TenantAllowBlockList"
$RecordTypes+="ConsumptionResource","HealthcareSignal","DlpImportResult","CDPCompliancePol icyExecution","MultiStageDisposition","PrivacyDataMatch"
$RecordTypes+="FilteringDocMetadata","FilteringEmailFeatures","PowerBIDlp","FilteringUrlInfo","FilteringAttachmentInfo","CoreReportingSettings"
$RecordTypes+="ComplianceConnector","PowerPlatformLockboxResourceAccessRequest","PowerPlatformLockboxResourceCommand","CDPPredictiveCodingLabel"
$RecordTypes+="CDPCompliancePolicyUserFeedback","WebpageActivityEndpoint","OMEPortal","CMImprovementActionChange","FilteringUrlClick"
$RecordTypes+="MipLabelAnalyticsAuditRecord","FilteringEntityEvent","FilteringRuleHits","FilteringMailSubmiss ion","LabelExplorer","MicrosoftManagedServicePlatform"
$RecordTypes+="PowerPlatformServiceActivity","ScorePlatformGenericAuditRecord","FilteringTimeTravelDocMetadata","Alert","AlertStatus"
$RecordTypes+="AlertIncident","IncidentStatus","Case","CaseInvestigation","RecordsManagement","PrivacyRemediation","DataShareOperation"
$RecordTypes+="CdpDlpSensitive","EHRConnector","FilteringMailGradingResult","PublicFolder","PrivacyTenantAuditHistoryRecord","AipScannerDiscoverEvent"
$RecordTypes+="EduDataLakeDownloadOperation","M365ComplianceConnector","MicrosoftGraphDataConnectOperation"

#Target SharePointOnline site and path (create the document library folder beforehand)
$global:siteUrl="https://xxxx.sharepoint.com/sites/DLPLogs/"
$global:targeturl ="/sites/DLPLogs/Shared Documents/Logs/"

#Generate credentials
$global:Credential = Get-AutomationPSCredential -Name "Office 365"

function GetLogandUpload{
Param (
[string]$RecordType, [string]$Operations=""
)
"Get logs: "+$RecordType
#Generate a unique session ID string with date and time
$sessionId=$RecordType+(Get-Date -Format "yyyyMMdd-HHmm")

#Retrieve logs in a loop of up to 5,000 x 10 times
$output=@();
for($i = 0; $i -lt 10; $i++){ 
if($Operations -ne $null -and $Operations.Length -ne 0) 
{ 
$result=Search-UnifiedAuditLog -RecordType $RecordType -Operations $Operations -Startdate $Startdate -Enddate $Enddate -SessionId $sessionId -SessionCommand ReturnLargeSet -ResultSize 5000 
} 
else 
{ 
$result=Search-UnifiedAuditLog -RecordType $RecordType -Startdate $Startdate -Enddate $Enddate -SessionId $sessionId -SessionCommand ReturnLargeSet -ResultSize 5000 
} 
$output+=$result 
"Query "+($i+1)+" round: "+$result.Count.ToString() + " results" 
if($result.count -ne 5000){break}
}
if($output.count -eq 0){ 
"No data" 
return 
}
"Total: "+$output.Count.ToString() + " results"

#Get the fields contained in the JSON from the first item for each operation type
$OperationTypes=$output|Group-Object Operations
$FieldName=@()
foreach($Operation in $OperationTypes){
$JsonRaw=$Operation.Group[0].AuditData|ConvertFrom-Json
$FieldsInJson=$JsonRaw|get-member -type NoteProperty
foreach($f in $FieldsInJson){
if($FieldName.Contains($f.Name) -eq $false) { $FieldName+=$f.Name}
}
}

#Generate a ScriptBlock to parse JSON for use with Select-Object
$Fields="ResultIndex", "CreationDate", "UserIds", "Operations", "RecordType"
foreach($f in $FieldName){
$sb1=[scriptblock]::Create('$JsonRaw.'+$f)
$sb2=[scriptblock]::Create('$att=$JsonRaw.'+$f+';if($att.GetType().Name -eq "Object[]" -or $att.GetType().Name -eq "PSCustomObject"){ConvertTo-Json -Compress -Depth 10 $att} else {$att}')
if($f -ne "RecordType") {$Fields+=@{Name=$f;Expression=$sb2}}
else {$Fields+=@{Name="RecordType2";Expression=$sb1}}
}

#Parse JSON and convert it to CSV format
$csv=@();
foreach($row in $output){
$JsonRaw=$row.AuditData|ConvertFrom-Json
$data=$row|Select-Object -Property $Fields
$csv+=$data
}

#Output
$outfile="C:\Report\"+$RecordType+".csv"
$csv|Export-Csv -Path $outfile -NoTypeInformation -Encoding UTF8

$fs = new-object System.IO.FileStream($outfile, [System.IO.FileMode]::Open, [System.IO.FileAccess]::Read)
[Microsoft.SharePoint.Client.File]::SaveBinaryDirect($global:ctx, $global:targeturl+$RecordType+".csv", $fs, $true)
$fs.Close()
"Upload completed."
}

#Main Processing
Connect-ExchangeOnline -credential $global:Credential

#Load CSOM Assemblies
Load-SPOnlineCSOMAssemblies
$global:ctx = New-Object Microsoft.SharePoint.Client.ClientContext($global:siteUrl)
$cre=$null
$count=0
while($cre -eq $null -and $count -lt 10){
$count++
try{$cre = New-Object Microsoft.SharePoint.Client.SharePointOnlineCredentials($Credential.UserName,$Credential.Password)}
catch{ 
"SPO Authentication Error" 
$_.Exception.Message 
Start-Sleep -s 5}
}
$global:ctx.Credentials=$cre

#Do Loop
foreach($r in $RecordTypes){ 
GetLogandUpload($r)
}

#Close
$ctx.Dispose()
Disconnect-ExchangeOnline -Confirm:$false
````