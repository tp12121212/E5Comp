## May become unavailable in the future
This script uses legacy tokens that allow access to all the functionality of the previous MDA. The recommended OAuth 2.0 tokens will have stricter authorization scope, and the APIs used in this script will no longer work. Therefore, this script may no longer work in the future. Reference: [MDA Token](https://learn.microsoft.com/en-us/defender-cloud-apps/api-authentication)

## About Throttling
MDA has a limit of 30 calls per minute. Depending on the MDA environment, API calls that exceed this limit can behave in two ways: asynchronously, where an immediate throttled error is returned, or synchronously, where a one-minute wait for a response is made and the correct result is returned. This script was tested in a synchronous environment, so error handling in asynchronous environments has not been properly verified.

# Retrying a governance action to apply a label in Defender for Cloud Apps
This sample script retrieves information from the Microsoft 365 Defender [Governance Log](https://security.microsoft.com/cloudapps/governance-log) using API access via PowerShell and retries any failed sensitivity label applications.
Specifically, it retrieves up to 300 labeling actions, both successful and failed, from the governance log over the past 8 hours and retries all except the following. Please adjust the number of governance log entries retrieved and the range of logs to exclude depending on the batch execution frequency.
- Operations related to files that have already been successfully labeled in the new log
- Operations that failed due to the label already being applied
- Operations that failed within the last hour
- Files created more than 24 hours ago
- Duplicate retries for the same file (Updated December 25, 2023)

For batch execution, place this script on Azure Automation or schedule it to run on a Windows machine with internet access and PowerShell running. (No special modules need to be installed.)
---(Updated December 25, 2023)---
Note: In the governance log, ShouldRetry = false is displayed when labeling fails or results in an internal error due to a Box download disable lock or various other issues. However, this has been changed to retry these cases (the ShouldRetry condition has been commented out with a #). Also, if there are multiple failed labeling logs for the same file, a single script execution will no longer retry labeling multiple times.
## Defender for Cloud Apps labeling behavior
Defender for Cloud Apps labeling works as follows:
1. The creation or upload of a new file is confirmed in the log as an activity.
1. A new file is recognized on the Files page.
1. If the file policy is met, a governance log entry for the file is recorded, and a labeling attempt is made as a governance action.
1. If the initial attempt fails, for example, because the file is being edited, the governance action is put into a pending state.
1. After the initial attempt, three retries are made at 15-minute intervals. If all four attempts fail within approximately 45 minutes, the governance action is put into a failed state.
(If the file is being edited continuously, the action is put into a failed state with the error "Failed to upload protected file.")
1. Once a governance action has failed, labeling will not be retried even if the file is subsequently updated.
1. You can retry a file that has failed labeling by manually retrying it from the governance log or by using this script.

## Preparation
### API Token
Defender for Cloud Apps [API] Visit the [Tokens](https://security.microsoft.com/cloudapps/settings?tabid=apiTokens) page and obtain a 64-character alphanumeric token in advance. Please note that once obtained, the token will not be displayed again, and if you forget the previous token, you will need to reissue it.

## Script body
````
#Parameters
#should be replaced by the tenant domain
$Uri="https://xxxxx.portal.cloudappsecurity.com/api/v1/governance/"

#64 chacters, should be obtained via MDA API Token page
$Token="xxxxxxxxx"

#Filter for finding governance actions related to labeling within last 8 hours
$t=[datetimeoffset]::Now.AddHours(-8).ToUnixTimeMilliseconds()
$filter='{"status":{"eq":[true, false]},"timestamp":{"gte":'+$t+'},"type":{"eq":["RmsProtectTask"]}}'

#Amount of governance actions to retrieve
$ResultSetSize=300

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
"sortField"="timestamp" 
"sortDirection"="desc" 
} 
do { 
$retryCall = $false 
try { 
$res=Invoke-RestMethod -Uri $Uri -Method "Post" -Headers $headers -Body $Body 
"Loop: $i, From " +$i*$batchSize +", " + $res.data.Count +" items" 
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
"Retrieved " +$output.count+" actions"

$completed=@()
$retried=@()
Foreach($d in $output){ 
$skipmessage=@() 
#Skip labeled file 
if($completed.IndexOf($d.targetObjectId) -ne -1){ 
$skipmessage+="Label file" 
} 
#Skip retried file 
if($retried.IndexOf($d.targetObjectId) -ne -1){ 
$skipmessage+="Retried file" 
} 
#Skip success 
if($d.status.isSuccess){ 
$completed+=$d.targetObjectId 
$skipmessage+="Successful task" 
} 
#Skip which is not supported to be retried #removed this condition #if($d.status.shouldRetry -eq $false){$skipmessage+="ShouldNotRetry"} 

#Skip files protected by any solution outside of MDA 
if($d.status.statusMessage.Contains("already protected")){ 
$completed+=$d.targetObjectId 
$skipmessage+="Already Protected" 
} 

#Skip actions which was taken within last 1 hour 
$initiated=([datetimeoffset]::FromUnixTimeMilliseconds($d.timestamp)).UtcDateTime 
If($initiated -ge (Get-Date).ToUniversalTime().AddHours(-1)){$skipmessage+="Within last 1 hours"} 

#Skip files created over 24 hours ago 
If($d.created.ToDateTime($null) -le (Get-Date).AddHours(-24)){$skipmessage+="Old file"} 

if($skipmessage.count -ge 1){ 
$d.targetObject+","+$d.targetObjectId+",skipped, due to " + ($skipmessage -join ", ") 
continue 
} 

$d.targetObject+","+$d.targetObjectId+",Retry" 
#Retry should be called with governance log id 
$RetryUri=$Uri+$d._id+"/retry/" 
$res2=Invoke-RestMethod -Uri $RetryUri -Method "Get" -Headers $headers 
$retried+=$d.targetObjectId 
Start-Sleep-Seconds 1
}

````