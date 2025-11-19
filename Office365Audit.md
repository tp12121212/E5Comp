# Outputting Office 365 Audit Logs
This is technical information about Office 365 audit logs. For information on the types of logs recorded in Office 365 audit logs and their retention, please also see the page [here](https://github.com/YoshihiroIchinose/Office365Audit/blob/main/RecordTypes.md).

## Maximum Data Retrieval with Search-UnifiedAuditLog
Regarding the maximum number of logs that can be retrieved with Search-UnifiedAuditLog, [Docs](https://docs.microsoft.com/ja-jp/powershell/module/exchange/search-unifiedauditlog?view=exchange-ps) suggests that specifying options such as -SessionCommandRetrunLargeSet allows you to retrieve up to 50,000 items of data in a single query. However, if you try it, you'll find that even with this option, specifying a value greater than 5,000 for -ResultSize will result in an error; if you don't specify -ResultSize, you'll only get 100 results. **In the end, the maximum amount of data that can be retrieved with a single command is still 5,000, the maximum for -ResultSize**. However, if you specify a session ID and the -SessionCommandRetrunLargeSet option, up to 50,000 results can be internally reserved. Therefore, by repeatedly executing the cmdlet with the same session ID using the same search criteria, you can retrieve a maximum dataset of 50,000 results, divided into 10 runs of 5,000 results each.** Conversely, with search settings spanning a wide range (more than 50,000 results), it's important to note that even if you run the same cmdlet multiple times and retrieve 5,000 results each time, you won't be able to retrieve all the logs.

## Handling AuditData
The Search-UnifiedAuditLog cmdlet can retrieve various types of Microsoft 365 logs. The available log types are summarized in the [Docs](https://docs.microsoft.com/ja-jp/office/office-365-management-api/office-365-management-activity-api-schema#auditlogrecordtype). However, because the data recorded varies depending on the log type, meaningful data is stored in JSON format in a column called AuditData. Therefore, outputting log details in CSV format or other formats can be cumbersome, as you must individually specify the values ​​to extract from the JSON according to the log type (operation for each record type). The PowerShell sample presented here first extracts one log per operation type from the retrieved logs and then analyzes the JSON attribute names contained within them in advance. This allows comprehensive output of data in CSV format without specifying individual attribute names. This example also uses Endpoint DLP logs, which extract device-side operations, as an example. This is likely to be in high demand in the future (requires an M365 E5, M365 E5 Compliance, or Infomatino Protection & Governance license and device onboarding).

## Exporting JSON in AuditData (Updated January 17, 2022)
You can use ConvertFrom-Json to parse JSON-formatted text data and extract it as an object. However, if the extracted attribute contains a value that includes an array, simply outputting it as text will result in that portion being expressed as System.Object[], and not all of the data will be exported. Therefore, when exporting JSON values ​​as text again, we use ConvertTo-Json -Compress -Depth 10 $attribute to convert the JSON values ​​to JSON format without line breaks at multiple levels and output the text. This allows you to export detailed information, such as confidential information matches in DLP Endpoint.

# PowerShell script
## Launch PowerShell with administrative privileges and run the following once beforehand.
```
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
Install-Module -Name ExchangeOnlineManagement
```
## Collect Endpoint DLP logs
```
# Connection ID/Password
$Id="xxx@xxx.onmicrosoft.com"
$Password = "xxxxx"
# Variable
$Startdate="2021/06/01"
$Enddate="2021/12/31"
$RecordType="DLPEndpoint"
$OutputFolder=[System.Environment]::GetFolderPath("Desktop")+"\"

# Credential generation
$SecPass=ConvertTo-SecureString -String $Password -AsPlainText -Force
$Credential= New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $Id, $SecPass

Import-Module ExchangeOnlineManagement
Connect-ExchangeOnline -credential $Credential

#Generate a unique session ID string using the date and time
$sessionId=$RecordType+(Get-Date -Format "yyyyMMdd-HHmm")

#Retrieve logs in a loop of up to 5,000 x 10 times
$output=@();
for($i = 0; $i -lt 10; $i++){
$result=Search-UnifiedAuditLog -RecordType $RecordType -Startdate $Startdate -Enddate $Enddate -SessionId $sessionId -SessionCommand ReturnLargeSet -ResultSize 5000
$output+=$result
"Query"+($i+1)+"th iteration: "+$result.Count.ToString() + "items retrieved"
if($result.count -ne 5000){break}
}
"Total: "+$Output.Count.ToString() + " Get items"

#Get fields contained in JSON from the first item for each operation type
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
$csv|Export-Csv -Path ($OutputFolder+$sessionId+".csv") -NoTypeInformation -Encoding UTF8

```
## (Bonus) Convert a CSV file to an Excel file with table formatting via Excel
```
$excel = new-Object -com excel.application
$excel.visible = $false
$book = $excel.Workbooks.open($OutputFolder+$sessionId+".csv")
$book.ActiveSheet.ListObjects.Add(1,$book.ActiveSheet.Range("A1").CurrentRegion ,$null,1).Name = "Table1"
$book.SaveAs($OutputFolder+$sessionId+".xlsx",51)
$book.close()
$excel.quit()

```
Note For 15,145 data items, a file that was 8.31 MB in CSV format was compressed to 2.68 MB, about one-third the size, by converting it to XLSX format.

## (Reference) Operation types confirmed in DLPEndpooint RecordType
(There may also be other Bluetooth operations)
- FileDeleted
- RemovableMediaUnlockedmount
- FileModified
- RemovableMediaMount
- FileUploadedToCloud
- FileCopiedToRemoteDesktopSession
- FileArchived
- FileCreated
- FileRead
- FileCopiedToNetworkShare
- FileCreatedOnNetworkShare
- FileCreatedOnRemovableMedia
- FileRenamed
- FileCopiedToRemovableMedia
- FileDownloadedFromBrowser
- FilePrinted
- FileMoved
- FileCopiedToClipboard

## (Reference) Attributes in the JSON included in the AuditData of the above Operation log
(Information included varies depending on the type of operation)
- Application
- ClientIP
- CreationTime
- DeviceName
- FileExtension
- FileSize
- FileType
- Hidden
- Id
- MDATPDeviceId
- ObjectId
- Operation
- OrganizationId
-Platform
-RecordType
-Scope
-SourceLocationType
-UserId
-UserKey
-UserType
- Version
-Workload
-RemovableMediaDeviceAttributes
-DestinationLocationType
- EnforcementMode
-OriginatingDomain
-RMSEncrypted
- SensitiveInfoTypeData
-Sha1
-Sha256
-TargetFilePath
-TargetDomain
-PolicyMatchInfo
-TargetPrinterName
-ParentArchiveHash
- SensitivityLabelEventData
- PreviousFileName