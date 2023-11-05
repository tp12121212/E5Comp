# Defender for Cloud Apps でのラベル適用のガバナンス アクションをリトライする
本サンプル スクリプトは、Microsoft 365 Defender の[ガバナンス ログ](https://security.microsoft.com/cloudapps/governance-log)の情報を、
PowerShell を使った API アクセスで取得し、失敗したままになっている秘密度ラベル適用をリトライするものです。
具体的には、ガバナンス ログの中から成功・失敗含めてラベル付けのアクションを最新 300 件取得し、そのうち以下のものを除いてリトライを実施します。
- 新しいログで既にラベル付けに成功しているファイルに関する操作
- リトライすべきではないというステータスのもの
- 既にラベルが付けらていて失敗しているもの
- 8 時間以上経過しているもの
- 1 時間以内に失敗しているもの

バッチ実行に当たっては、本スクリプトを、
Azure Automation 上に配置し、スケージュール実行します。

## 事前準備
### API トークン
Defender for Cloud Apps の [API トークン](https://security.microsoft.com/cloudapps/settings?tabid=apiTokens)のページにアクセスし、
64 文字の英数字で構成されるトークンを事前に取得しておきます。取得後は再表示されず、以前のトークンが分からなくなった場合には、トークンの再発行が必要となる点に注意します。

## スクリプト本体
````
#Parameters
#should be replaced by the tenant domain
$Uri="https://xxxxx.portal.cloudappsecurity.com/api/v1/files/"

#64 chacters, should be obtained via MDA API Token page
$Token="xxxxxxxxx"

#Filter for finding protected files in SPO withoug sensitivity labels
$filter='{"status":{"eq":[true, false]},"taskName":[RmsProtectTask]}'

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
			"Loop: $i, From " +$i*$batchSize
			$res=Invoke-RestMethod -Uri $Uri -Method "Post" -Headers $headers -Body $Body
			}
		catch {
			if ($_ -like 'The remote server returned an error: (429) TOO MANY REQUESTS.'){
				$retryCall = $true
				Start-Sleep -Seconds 5
			}
			ElseIf ($_ -match 'throttled'){
				$retryCall = $true
				Start-Sleep -Seconds 60
			}
			ElseIf ($_ -like '504' -or $_ -like '502'){
				$retryCall = $true
				Start-Sleep -Seconds 5
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

$completed=@()
Foreach($d in $res.data){
        $skipmessage=@()
	#Skip labeled file
	if($completed.IndexOf($d.targetObjectId) -ne -1){
		$skipmessage+="Labeld file"
	}
	#Skip success
	if($d.status.isSuccess){
		$completed+=$d.targetObjectId
		$skipmessage+="Successful task"
	}
	#Skip which is not supporsed to be retried
	if($d.status.shouldRetry -eq $false){$skipmessage+="ShouldNotRetry"}
	#Skip protected files
	if($d.status.statusMessage.Contains("already protected")){$skipmessage+="Already Protected"}
	#Skip files which was created over 8 hours before
	if($d.created.ToDateTime($null) -le (Get-Date).AddHours(-8)){$skipmessage+="Older than 8 hours"}
	#Skip files which was created within last 1 hour
	if($d.created.ToDateTime($null) -ge (Get-Date).AddHours(-1)){$skipmessage+="Within last 1 hours"}
	if($skipmessage.count -gt 1){
        	$d.targetObject+","+$d.targetObjectId+",skipped, due to " + ($skipmessage -join ", ")
        	continue
        }
	$d.targetObject+","+$d.targetObjectId+",Retry"
	#Retry should be called with governance log id
	$RetryUri=$Uri+$d._id+"/retry/"
	$res2=Invoke-RestMethod -Uri $RetryUri -Method "Get" -Headers $headers
}

````