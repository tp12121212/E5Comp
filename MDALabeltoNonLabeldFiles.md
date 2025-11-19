## May become unavailable in the future
This script uses legacy tokens that allow access to all the functionality of the previous MDA. The recommended OAuth 2.0 tokens will have stricter authorization scope, and the APIs used in this script will no longer work. Therefore, this script may no longer work in the future. Reference: [MDA Token](https://learn.microsoft.com/en-us/defender-cloud-apps/api-authentication)

## About Throttling
MDA has a limit of 30 calls per minute. Depending on the MDA environment, API calls that exceed this limit can behave in two ways: asynchronously, where an immediate throttled error is returned, or synchronously, where a one-minute wait for a response is made and the correct result is returned. This script was tested in a synchronous environment, so error handling in asynchronous environments has not been properly verified.

# Assigning sensitivity labels to files in a specific folder using the Defender for Cloud Apps API
This script is a sample script that uses the Defender for Cloud Apps API to assign sensitivity labels to unlabeled files in a specific folder.
While the same effect can be achieved using governance actions in file policies, this script is intended as a supplement to manual labeling operations in cases where governance actions are not performed due to an error or other reason.

<br/>
<br/>
As a prerequisite, the target folder and file must be visible in Defender for Cloud Apps, the file must be a file type that supports sensitivity labeling, and it must not yet have a label assigned. <br/>
<br/>
Using the same mechanism in principle, you can also perform actions such as removing a label or unsharing by replacing the action part.

### Settings required for each environment (Specify the variables in the script with the following environment-specific values.)
#### 1. 64-character Defender for Cloud Apps token
Token value obtained from the [API Token](https://security.microsoft.com/cloudapps/settings?tabid=apiTokens) page<br/>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/MDA_Autolabel4.png" width="50%">
#### 2. Defender for Cloud Apps API URL
URL displayed when obtaining the token (To use the labeling API, you must also include a regional domain such as .us3).<br/>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/MDA_Autolabel3.png" width="50%">
#### 3. The instance number of the target app.
The 5-7 digit number following /service-app/ in the URL of the app's page, which appears when you click the app from the Files page, etc.
<br/>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/MDA_Autolabel1.png" width="50%">
#### 4. File ID of the target folder
The "File ID" value in the details of the target folder on the Files page.

You can also check the folder's "File ID" value by selecting "Show Hierarchy" from the details of the target file to display the folder's details.

SPO/OD4B For example, "736e3abc-13d1-44fe-9aed-3b56f9878ead|d286c00e-de8e-4eb1-9881-61bd97a69abc"<br/>
For Box, "239616417475"<br/>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/MDA_Autolabel2.png" width="50%"> <br/>
In the script, you can specify multiple folders separated by commas, such as $folders=@("xxxx"), for example, $folders=@("A","B","C")
#### 5. File IDs of subfolders to exclude from labeling (optional)
You can specify subfolders within the folder specified in 4 to exclude from labeling. <br/>
Check and specify the "File ID" value of the subfolder you want to exclude using the same method as in step 4. <br/>
Note that subfolders within the excluded subfolder will also be excluded. <br/>
In the script, you can specify multiple folders separated by commas, such as $excludedSubfolders=@(""), for example, $excludedSubfolders=@("A","B","C")
#### 6. Display name of the sensitivity label to assign

### Items to adjust as needed
1. Number of items to retrieve per query (This script retrieves 100 items in descending order by modification date)
2. Modification date range of files to label (This script targets files updated between 1 and 24 hours ago)

## What this script does
1. Get a list of sensitivity labels and obtain their IDs
2. Recursively query subfolders within the specified folder to identify the target subfolders
3. Retrieve all files directly under the target folder whose modification date falls within the target range and that do not have a sensitivity label.
4. Trigger the labeling operation on the target files.

### Script body
````
#<--Parameters
#Should be replaced by the tenant domain and URL, which can be found when you get an MDA API Token
$baseUrl="xxxxxx.us3.portal.cloudappsecurity.com"

#64 parameters, should be obtained via the MDA API Token page
$Token="xxxxxxxxxxxxx"

#Label name to apply
$labelname="Confidential"

#Targeted app instance, which can be identified in a URL string as a 5-digit number after "/service-app/" when you click an app name on the MDA File page
$instance="20892"

#Targeted folders as an array, which can be identified on the MDA File page as a "File ID" of the folder
#Folder in SPO/OD4B is like "736e3abc-13d1-44fe-9aed-3b56f9878ead|d286c00e-de8e-4eb1-9881-61bd97a69abc"
#Folder in Box is like "239616417475"
$folders=@("736e3762-13d1-44fe-9aed-3b56f9878ead|d286c00e-de8e-4eb1-9881-61bd97a608e3")

#Subfolders which should be excluded for labeling (Same ID format as that of a target folder)
$excludedSubfolders=@("")

#scope of target files specified by time range of modified data and the number of files
$s=[datetimeoffset]::Now.AddHours(-24).ToUnixTimeMilliseconds()
$e=[datetimeoffset]::Now.AddHours(-1).ToUnixTimeMilliseconds()
#Parameters-->

$ResultSetSize=100

#Global variables
$targetFiles=@()
$targetFolders=@()
$pathInfo=@{}
$headers=@{"Authorization" = "Token "+$Token}

Function RestCall ($u, $m, $h, $b=$null){#Foo all GET/POST REST call
$maxRetry=5
$retryCount=0
do{ 
$retryCall = $false 
$retryCount++ 
try{ 
$res=Invoke-RestMethod -Uri $u -Method $m -Headers $h -Body $b 
} catch{ 
if($_ -match 'throttled'){
$retryCall = $true 
"Throttled" 
Start-Sleep -Seconds 60 
} 
else { 
"Error url: " +$u 
throw $_ 
} 
}#end of Catch 
}#end of do loop 
while ($retryCall -and $retryCount -le $MaxRetry) 

if($retryCount -ge $MaxRetry){ 
$u+": Retry count exceeded limit." 
exit 
} 
return $res
}

Function GetLabel($labelName){#Get the label ID by a label name 
$Uri="https://"+$global:baseUrl+"/api/v1/get_rms_encryption_labels/" 
$res=RestCall $Uri "Get" $global:headers 
foreach($l in $res.data){ If($l.name.equals($labelName)){ 
return $l.id 
} 
} 
return $null
}

Function GetFoldersRecursive($parent){#Get folders recursively under a spcified folder 
"Get folders from " + $parent 
$filter='{"parentFolder":{"eq":["'+$parent+'"]},"fileType":{"eq":[6]},"instance":{"eq":['+$global:instance+']}}' 
$batchSize=100 
$Uri="https://"+$global:baseUrl+"/api/v1/files/" 

$loopcount = [int][Math]::Ceiling($global:ResultSetSize / $batchSize) 
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
$res=RestCall $Uri "Post" $global:headers $Body 
if($res -eq $null){ 
"Error, No data returned" 
return 
} 
"Loop: $i, From " +$i*$batchSize +", " + $res.data.Count +" folders" 
$output+=$res.data 
if($res.data.Count -lt $batchsize){break} 
} 
"Retrieved " +$output.count+" folders" 
foreach($item in $output){ 
if($global:excludedSubFolders.Contains($item.id)){ 
"Subfolder: " + $item.name +" was excluded." 
continue 
} 
if($item.isFolder){ 
$global:targetFolders+=$item._id 
GetFoldersRecursive($item._id) 
} 
}
}

Function GetFolderPath($_id){#Get a file path for the file and store it to the associative array 
$Uri="https://"+$global:baseUrl+"/api/v1/files/"+$_id+"/effective_parents/" 
$res=RestCall $Uri "Get" $global:headers 
$path=@() 
foreach($p in $res.data){ 
$path+=$p.name 
} 
Return "/"+[string]::Join("/", $path)
}

Function GetFolderItems($parent){#Get recent files directly under a spcified folder 
"Get files from " + $parent 
$filter='{"modifiedDate":{"range":[{"start":'+$global:s+',"end":'+$global:e+'}]},"fileType":{"neq":[6]},"fileLabels":{"isnotset":true},' 
$filter+='"parentFolder":{"eq":["'+$parent+'"]},"instance":{"eq":['+$global:instance+']}}' 
$batchSize=100 
$Uri="https://"+$global:baseUrl+"/api/v1/files/" 

$loopcount = [int][Math]::Ceiling($global:ResultSetSize / $batchSize) 
$output=@() 
For($i=0;$i -lt $loopcount; $i++){ 
$limit=$batchSize if($loopcount -1 -eq $i){$limit=$ResultSetSize % $batchSize} 
if($limit -eq 0){$limit=$batchSize} 

$Body=@{ 
"skip"=0 + $i*$batchSize 
"limit"=$limit 
"filters"=$filter 
"sortField"="modifiedDate" 
"sortDirection"="desc" 
} 

$res=RestCall $Uri "Post" $global:headers $Body 
if($res -eq $null){ 
"Error, No data returned" 
return 
} 
"Loop: $i, From " +$i*$batchSize +", " + $res.data.Count +" items" 
$output+=$res.data 
if($res.data.Count -lt $batchsize){break} 
} 
"Retrieved " +$output.count+" files" 
foreach($item in $output){ 
Foreach($act in $item.actions){ 
If($act.task_name -eq "RmsProtectTask"){#Find files which have a RmsProtectTask action 
$global:pathInfo[$item._id]=(GetFolderPath $item._id) 
"Found non-labeled file " + $global:pathInfo[$item._id] 
$global:targetFiles+=$item 
break 
} 
} 
}
}

#Get the label id by a specified label name
$label=GetLabel($labelname)
if($label -eq $null){ 
"Error. The spcified Label is not found." 
exit
}

#Get all subfolders under specified folders
foreach ($f in $folders){ GetFoldersRecursive($f)
}
$allFolders=$folders+$targetFolders
"---------------"
"Total folders: "+$allFolders.count
" "
#Get all files without a label which can be labeled in targetfolders
foreach ($f in $allFolders){ 
GetFolderItems($f)
}
"---------------"
"Total files to be labeled: "+$targetFiles.count
" "

#Kick maunal labeling for all targeted files
foreach($f in $targetFiles){ 
$Uri="https://"+$global:baseUrl+"/api/v1/files/bulk_governance/" 
#Prepare request body as a text because its order matters 
$Body='{"task_name":"RmsProtectTask",' $Body+='"entities":[{"id":"'+$f._id+'","appId":'+$f.appId+'}],'
$Body+='"params":{"labelId":"'+$label+'"}}'
"Apply the label to: " + $global:pathInfo[$f._id]
RestCall $Uri "Post" $global:headers $Body
}

````
## Output of execution results on Azure Automation
### File on Box
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/MDA_AutoLabel.png" width="50%">

### File on SPO
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/MDA_Autolabel5.png" width="50%">

### Labeling History
The results and history of labeling operations can be viewed in the Governance Log.
### File on SPO
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/MDA_Autolabel6.png" width="50%">