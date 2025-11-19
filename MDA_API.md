## May become unavailable in the future
This script uses legacy tokens that allow access to all the functionality of the previous MDA. The recommended OAuth 2.0 tokens will have stricter authorization scope, and the APIs used in this script will no longer work. Therefore, this script may no longer work in the future. Reference: [MDA Token](https://learn.microsoft.com/en-us/defender-cloud-apps/api-authentication)

## About Throttling
MDA has a limit of 30 calls per minute. Depending on the MDA environment, API calls that exceed this limit can behave in two ways: asynchronously, where an immediate throttled error is returned, or synchronously, where a one-minute wait for a response is made and the correct result is returned. This script was tested in a synchronous environment, so error handling in asynchronous environments has not been properly verified.

# Retrieve information from the Defender for Cloud Apps File page via API
This sample script retrieves information from the Microsoft 365 Defender File page via API access using PowerShell. The File page also allows you to set various filter conditions and view a list of files matching the conditions from SPO/OD4B and connected cloud storage. You can also retrieve data in CSV format from the UI using the Export function, but there is a limit of 5,000 items. When accessing via API, you can retrieve information on more than 5,000 files matching the conditions by repeatedly retrieving data for up to 100 items.
## Preparation
### API Token
Access the Defender for Cloud Apps [API Token](https://security.microsoft.com/cloudapps/settings?tabid=apiTokens) page and obtain a 64-character alphanumeric token in advance. Please note that the token will not be displayed again after acquisition, and if you lose track of the previous token, you will need to reissue it.
### Obtaining Filter Settings Showing Search Criteria
To retrieve files, you must set the filter criteria to be used for narrowing down the results. When you filter on the Microsoft 365 Defender [Files page](https://security.microsoft.com/cloudapps/files), the filter criteria are embedded in the URL. Define them in JSON format, referring to the example script and confirming which attributes and values ​​are filtered based on the criteria. You can also directly view the JSON-formatted filter string by using your browser's developer tools on the Microsoft 365 Defender [Files page](https://security.microsoft.com/cloudapps/files) and viewing the contents of the POST request to https://security.microsoft.com/apiproxy/mcas/cas/api/v1/files/count/. The condition in the example script, service 20892, narrows the search to files on SharePoint Online. Other general-purpose PowerShell cmdlets and sample code for Defender for Cloud Apps can be found on [this page](https://github.com/microsoft/MCAS).
### Filter Examples
Files on SharePoint Online encrypted without sensitivity labels
`$filter='{"fileScanLabels":{"eq":["^Azure RMS encrypted file"]},"service":{"eq":[20892]}}'`

Files shared with users in a specific domain
`$filter='{"service":{"eq":[20892]},"collaborators.withDomain":{"eq":["hotmail.com","gmail.com","icloud.com"]}}'`

Files on SharePoint Online with individual sensitivity labels
`$filter='{"service":{"eq":[20892]},"fileLabels":{"eq":["Individual Protection","Confidential"]}}'`

Files shared outside the company
`$filter='{"sharing":{"eq":[2,3,4]}}'`

## Script body
````
#Parameters
#should be replaced by the tenant domain
$Uri="https://xxxxx.portal.cloudappsecurity.com/api/v1/files/"

#64 chacters, should be obtained via MDA API Token page
$Token="xxxxxxxxx"

#Filter for finding protected files in SPO withoug sensitivity labels
$filter='{"fileScanLabels":{"eq":["^Azure RMS encrypted file"]},"service":{"eq":[20892]}}'

#Filter for finding files in SPO shared with some specific domain users
#$filter='{"service":{"eq":[20892]},"collaborators.withDomain":{"eq":["hotmail.com","gmail.com","icloud.com"]}}'

#Filter for finding files in SPO with some specific sensitivity labels
#$filter='{"service":{"eq":[20892]},"fileLabels":{"eq":["Personal Protection","Confidential"]}}'

#Filter for finding files shared externally
#$filter='{"sharing":{"eq":[2,3,4]}}'

#Amount of files to retrieve
$ResultSetSize=300

#ouput csv file
$OutputFile=[System.Environment]::GetFolderPath("Desktop")+"\MDAFiles.csv"

#Getting files
$batchSize=100 # the fixed limit for a single request
$loopcount = [int][Math]::Ceiling($ResultSetSize / $batchSize)
$headers=@{"Authorization" = "Token "+$Token}
$output=@()
For($i=0;$i -lt $loopcount; $i++){ 
$limit=$batchSize 
if($loopcount -1 -eq $i){$limit=$ResultSetSize % $batchSize} 
if($limit -eq 0){$limit=$batchSize} 
$Body=@{ 
"skip"=0 + $i*$batchSize 
"limit"=$limit 
"filters"=$filter 
"sortField"="modifiedDate" 
"sortDirection"="desc" 
} 
do { 
$retryCall = $false 
try { 
"Loop: $i, From " +$i*$batchSize $res=Invoke-RestMethod -Uri $Uri -Method "Post" -Headers $headers -Body $Body 
} 
catch { 
if ($_ -like 'The remote server returned an error: (429) TOO MANY REQUESTS.'){ 
$retryCall = $true 
Start-Sleep-Seconds 5 
} 
ElseIf ($_ -match 'throttled'){ 
$retryCall = $true 
Start-Sleep -Seconds 60 
} 
ElseIf ($_ -like '504' -or $_ -like '502'){ 
$retryCall = $true 
Start-Sleep-Seconds 5 
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

#Fields Adjustments as script blocks to remove actions, convert createDate/modifiedData into LocalDateTime, and expand recursive Json text
$FieldName= $output[0] |get-member -MemberType NoteProperty|foreach-object{$_.Name.ToString()}
$FieldName={$FieldName}.Invoke()
$FieldName.Remove("actions")
$Fields=@()
foreach($f in $FieldName){ 
if($f -eq "createdDate" -or $f -eq "modifiedDate"){ 
$sb=[scriptblock]::Create('$att=$_.'+$f+';$att=([Int64]$att)/1000;([datetimeoffset]::FromUnixTimeSeconds($att)).LocalDateTime') $Fields+=@{Name=$f.ToString();Expression=$sb} 
continue 
} 
$sb=[scriptblock]::Create('$att=$_.'+$f+';if($att.GetType().Name -eq "Object[]" -or $att.GetType().Name -eq "PSCustomObject"){ConvertTo-Json -Compress -Depth 10 $att} else {$att}') 
$Fields+=@{Name=$f.ToString();Expression=$sb} 
}

#Save as a csv file
$csv=@()
foreach($row in $output){ 
$csv+=$row|Select-Object -Property $Fields
}
$csv|Export-Csv -Path $OutputFile -NoTypeInformation -Encoding UTF8
````