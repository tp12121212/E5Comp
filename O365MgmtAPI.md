# Retrieving Audit Logs Using the Office 365 Management API
This is a sample of retrieving Office 365 audit logs from PowerShell using the Office 365 Management API, which supports a larger scale than Search-UnifiedAuditLog. As a preliminary step, you must register an app, set permissions, and obtain a Client Secret to use the API, as described here. When retrieving logs using the Office 365 Management API, you cannot specify detailed filtering criteria. Instead, you select one of five categories of Office 365 audit logs and retrieve logs within a specific time range. When you start a subscription for each of these five types of logs, a BLOB file containing a certain number of logs for each of the five types is gradually accumulated. A single query can retrieve the URL for a log BLOB covering a maximum 24-hour period going back up to seven days, and the contents of the log can be retrieved from that URL. The log retrieval itself is completed in steps 1-5 below, and to output to CSV, steps 6-9 retrieve all types of first-level columns from the entire log and output the values ​​of all columns for each log item. Step 10 includes an extra step for converting the data to a table in .xlsx format.

1. Obtain an access token using the Application ID and Client Secret (x1 Post request)
2. Check whether the subscription for the retrieved logs is valid (x1 Get request)
3. If not valid, start the subscription (x1 Post request)
4. Specify the target date range and obtain the URLs of the BLOB files for multiple logs (x1 Get request, but if there are many logs, it may be paginated and require multiple requests)
5. Access the URLs of the retrieved BLOB files one by one, retrieve their contents, and concatenate them (a GET request for each target BLOB file)
6. To output the retrieved logs in CSV format, identify all column types contained in the first item for each operation type.
7. When outputting the retrieved logs in CSV format, prepare a script block with a ConvertTo-Json call to ensure no values ​​are omitted.
8. To output the retrieved data in CSV format, output all column values ​​for the entire log for each item.
9. Output the retrieved data in CSV format.
10. Convert the CSV output file into a table. Save it as .xlsx.

Note that if there are more than 2,000 requests per minute, requests may result in errors depending on the scale. For more information, see [here](https://docs.microsoft.com/ja-jp/office/office-365-management-api/office-365-management-activity-api-reference#api-throttling).

#About DLP.ALL
According to this [Blog](https://techcommunity.microsoft.com/t5/security-compliance-and-identity/microsoft-365-compliance-audit-log-activities-via-o365/ba-p/2957297), the following logs are recorded in DLP.ALL, which overlap with those recorded in Audit.Exchange, Audit.SharePoint, and Audit.General. For example, DLPEndpoint logs can be obtained from both DLP.ALL and Audit.General.

|RecordType|Name|
| --- | --- |
|11|ComplianceDLPSharePoint|
|13|ComplianceDLPExchange|
|33|ComplianceDLPSharePointClassification|
|63|DLPEndpoint|
|99|OnPremisesFileShareScannerDlp|
|100|OnPremisesSharePointScannerDlp|

# PowerShell sample code
````
$AppClientID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx"
$ClientSecretValue = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
$TenantGUID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx"
$tenantdomain = "xxxxxx.onmicrosoft.com"
$OutputPath = "C:\Users\user01\Desktop\MGMTLog\"

$APIResource ="https://manage.office.com"
$BaseURI = "$APIResource/api/v1.0/$tenantGUID/activity/feed/subscriptions"

#Select one of the following five types
$Subscription = "Audit.General"
#$Subscription = "Audit.AzureActiveDirectory"
#$Subscription = "Audit.Exchange"
#$Subscription = "Audit.SharePoint"
#$Subscription = "DLP.All"

$today=(Get-Date)
#Logs can be retrieved up to 24 hours within the last 7 days.
#Retrieve one day's worth of data from 1-6 days ago in local time.
$daysdiff=3
$end=$today.AddDays(-$daysdiff+1).Date.ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ss")
$start = $today.AddDays(-$daysdiff).Date.ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ss")
$filter="startTime="+$start+"&endTime="+$end
$filename = ($OutputPath + $Subscription + "_" + $today.AddDays(-$daysdiff).Date.ToString("yyyy-MM-dd") + ".csv")

# 1. Obtain an access token
$body = @{grant_type="client_credentials";resource=$APIResource;client_id=$AppClientID;client_secret=$ClientSecretValue}
try{
$oauth = Invoke-RestMethod -Method Post -Uri "https://login.microsoftonline.com/$tenantdomain/oauth2/token?api-version=1.0" -Body $body -ErrorAction Stop
$OfficeToken = @{'Authorization'="$($oauth.token_type) $($oauth.access_token)"}
}
catch {
write-host -ForegroundColor Red "Invoke-RestMethod failed." + $error[0]
exit
}

# 2. Check subscription
$subs = Invoke-WebRequest -Headers $OfficeToken -Uri "$BaseURI/list" -UseBasicParsing |ConvertFrom-Json
$enabled=$false
foreach($sub in $subs){
if($sub.ContentType.ToLower() -eq $Subscription.ToLower() -and $sub.Status -eq "enabled"){$enabled=$true}
}

# 3. Create subscription if not enabled
if(!$enabled){
Invoke-WebRequest -Method Post -Headers $OfficeToken -Uri "$BaseURI/start?contentType=$Subscription" -UseBasicParsing -ErrorAction Stop
}

#4. Specify the target date range and obtain the URLs of multiple log BLOB files.
$output=""
$next="$BaseURI/content?contentType=$Subscription&PublisherIdentifier=$TenantGUID&$filter"
while ($next){
$repsonse=Invoke-WebRequest -Headers $OfficeToken -Uri $next -UseBasicParsing
$output += $repsonse.Content
$next = $response.Headers.NextPageUri
}
$uris = (ConvertFrom-Json $output).contentUri

#5. Access the URLs of the retrieved BLOB files one by one, retrieve their contents, and concatenate them.
$Logdata=@()
foreach($uri in $uris){
$Logdata+= Invoke-RestMethod -Uri $uri -Headers $Officetoken
}

#6. Extract the first 1 for each operation type to output in CSV format. Identify all column types contained in the first item.
$OperationTypes=$Logdata|Group-Object Operation
$FieldName=@()
foreach($Operation in $OperationTypes){
$FieldsinLog=$Operation.Group[0]|get-member -type NoteProperty
foreach($f in $FieldsinLog){
if(!$FieldName.Contains($f.Name)) { $FieldName+=$f.Name}
}
}

#7. When outputting the retrieved log to CSV, prepare a script block with a ConvertTo-Json operation to ensure values ​​are not omitted.
$Fields=@()
foreach($f in $FieldName){
$sb=[scriptblock]::Create('$att=$d.'+$f+';if($att.GetType().Name -eq "Object[]" -or $att.GetType().Name -eq "PSCustomObject"){ConvertTo-Json -Compress -Depth 10 $att} else {$att}')
$Fields+=@{Name=$f;Expression=$sb}
}

#To output retrieved data in CSV format, output all column values ​​for the entire log for each item.
$csv=@();
foreach($d in $Logdata){
$output=$d|Select-Object -Property $Fields
$csv+=$output
}

#9. Output retrieved data in CSV format
$csv|Export-Csv -Path $filename -NoTypeInformation -Encoding UTF8

#10. Convert the CSV output file to a table in .xlsx format. Save it as
$excel = new-Object -com excel.application
$excel.visible = $false
$book = $excel.Workbooks.open($filename)
$book.ActiveSheet.ListObjects.Add(1,$book.ActiveSheet.Range("A1").CurrentRegion ,$null,1).Name = "Table1"
$book.SaveAs($filename.Replace(".csv",".xlsx"),51)
$book.close()
$excel.quit()
````