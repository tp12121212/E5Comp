# Sample for Exporting Activity Explorer Logs to CSV
This sample uses the newly added Export-ActivityExplorerData command to retrieve up to 30 days' worth of logs, up to 50,000 rows (5,000 rows x 10 times). A standalone command
is limited to retrieving up to 5,000 rows per execution. Therefore, to retrieve logs exceeding 5,000 rows, you must specify the Watermark variable returned with the results as the next PageCookie
and perform multiple retrievals. Currently, column names are recorded even when output in CSV format, so there is no need to process the data, such as retrieving data in JSON format and then converting it to CSV, to obtain column names.

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
# Variables
$date=get-date
$e=$date.ToUniversalTime().ToString("yyyy/MM/dd HH:mm:ss")
$s=$date.AddDays(-30).ToUniversalTime().ToString("yyyy/MM/dd HH:mm:ss")
$OutputFile=[System.Environment]::GetFolderPath("Desktop")+"\AEJ.csv"
#$filter1=@("Activity", "DLPRuleMatch")
#$filter2=@("Workload", "Exchange")
#$filter3=@("Policy", "DLPPolicyName")

#Generate Credential
$SecPass=ConvertTo-SecureString -String $Password -AsPlainText -Force
$Credential= New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $Id, $SecPass

Connect-IPPSSession -credential $Credential

# Get logs up to 5,000 lines x 10 times
$output=@();
$watermark="";
for($i = 0; $i -lt 10; $i++){
$d=Export-ActivityExplorerData -OutputFormat csv -StartTime $s -EndTime $e -PageSize 5000 -PageCookie $watermark
#$d=Export-ActivityExplorerData -OutputFormat csv -StartTime $s -EndTime $e -PageSize 5000 -PageCookie $watermark -filter1 $filter1 -filter2 $filter2 -$filter3 $filter3
$output+=$d.ResultData
"Loop $i :"+$d.RecordCount+"rows"
if($d.LastPage){break;}
$watermark=$d.WaterMark
}

# Export to an Excel file
$output|out-file ($OutputFile) -Encoding UTF8
# Export only specific columns
# $output|ConvertFrom-Csv|Select-Object -Property "Activity", "Happened", "User", "FilePath", "TargetFilePath", "Application", "DeviceName", "Manufacturer", "SerialNumber", "Model" |Export-Csv ($OutputFile) -Encoding UTF8 -NoTypeInformation

```

## (Bonus) Exporting to CSV via Excel Convert a file to an Excel file with table formatting
```
$excel = new-Object -com excel.application
$excel.visible = $false
$book = $excel.Workbooks.open($OutputFile)
$book.ActiveSheet.ListObjects.Add(1,$book.ActiveSheet.Range("A1").CurrentRegion ,$null,1).Name = "Table1"
$book.SaveAs($OutputFile.replace(".csv",".xlsx"),51)
$book.close()
$excel.quit()
```
## Example of Excel output result
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AE2.png">