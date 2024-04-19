# Export sample of Activity Explorer logs to CSV
This is a sample of using the newly added Export-ActivityExplorerData command to retrieve 50,000 lines (5,000 lines x 10 times) of logs for up to 30 days. In the command of bare single execution,
There is a acquisition limit of up to 5,000 lines per time, so when acquiring logs exceeding 5,000 lines, specify the WaterMark variable returned together with the result as the next PageCookie,
It is necessary to acquire it divided into multiple times. Currently, even if it is output in CSV format, the column name will be recorded, so there is no need for data processing such as acquiring data in JSON format and converting it to CSV to get the column name.
 
# PowerShell Script
## Launch PowerShell with administrative authority and execute the following once in advance
```
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
Install-Module -Name ExchangeOnlineManagement
```
## Get Endpooint DLP logs collectively
```
#ID/password used for connection
$Id="xxx@xxx.onmicrosoft.com"
$Password = "xxxxx"
#Variable
$date=get-date
$e=$date.ToUniversalTime(). ToString("yyyy/MM/dd HH:mm:ss")
$s=$date.AddDays(-30). ToUniversalTime(). ToString("yyyy/MM/dd HH:mm:ss")
$OutputFile=[System.Environment]::GetFolderPath("Desktop")+"\AEJ.csv"
#$filter1=@("Activity", "DLPRuleMatch")
#$filter2=@("Workload", "Exchange")
#$filter3=@("Policy", "DLPPolicyName")

#Credential Generation
$SecPass =ConvertTo-SecureString -String $Password -AsPlainText -Force
$Credential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $Id, $SecPass

Connect-IPPSSession -credential $Credential

# Get logs up to 5,000 lines x 10 times
$output=@();
$watermark="";
for($i = 0; $i -lt 10; $i++){
    $d=Export-ActivityExplorerData -OutputFormat csv -StartTime $s -EndTime $e -PageSize 5000 -PageCookie $watermark
    #$d=Export-ActivityExplorerData -OutputFormat csv -StartTime $s -EndTime $e -PageSize 5000 -PageCookie $watermark -filter1 $filter1 -filter2 $filter2 -$filter3 $filter3
    $output+=$d.ResultData
        "Loop $i :"+$d.RecordCount+" line"
    if($d.LastPage){break;}
    $watermark=$d.WaterMark
}

# Export to an Excel file
$output|out-file ($OutputFile) -Encoding UTF8
```

## (extra) Convert CSV files to Excel files with table format through Excel
```
$excel = new-Object -com excel.application
$excel.visible = $false
$book = $excel.Workbooks.open($OutputFile)
$book.ActiveSheet.ListObjects.Add(1,$book.ActiveSheet.Range("A1"). CurrentRegion ,$null,1). Name = "Table 1"
$book.SaveAs($OutputFile.replace(".csv",".xlsx"),51)
$book.close()
$excel.quit()
```
## Example of Excel output results
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AE2.png">