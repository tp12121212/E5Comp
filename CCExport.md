# Bulk Download Content Matched by Communication Compliance Policies
## Overview
In Communication Compliance, a mailbox is created for each policy. By performing a content search on this mailbox, you can download all messages that match the policy in .pst format.

Here, we introduce a sample script that automates the process of extracting messages that match specific Communication Compliance policies.

## How It Works
By using the -AllowNotFoundExchangeLocationsEnabled $true parameter, a content search can also search hidden mailboxes in these systems.
This content search extracts messages sent after a specific date, but you can change the scope of the search by modifying the search criteria as needed.
Performing a content search also requires appropriate permission settings. <br>
[Reference: Assigning eDiscovery Permissions in the Compliance Portal](https://learn.microsoft.com/ja-jp/microsoft-365/compliance/assign-ediscovery-permissions?view=o365-worldwide)

## About the Scope of Automation
The Content Search mechanism involves the following steps: defining the search, starting the search, waiting for the search to complete, starting the export, waiting for the export to complete, and downloading the export results.
However, the final step of downloading the export results cannot be completed using PowerShell alone; it requires interactive support through the ClickOnce .NET app. This script automates the process up to launching the ClickOnce .NET app. After launching the ClickOnece app, paste the export key copied to the clipboard, specify the download location, and begin the download.
The downloaded .pst file can be loaded into Outlook by selecting Add from the Outlook File menu > Account Settings > Data Files. <br>
[Reference: Exporting a Content Search Report](https://learn.microsoft.com/ja-jp/microsoft-365/compliance/export-a-content-search-report?view=o365-worldwide)

## Script Sample
````
#Specify the CC policy name
$CCPolicyName="CCPolicy"

#Search criteria
$Query="sent>=2022-11-01T00:00:00+09:00"

$SearchName=$CCPolicyName+" at "+(Get-Date -Format "yyyy-MM-dd HHmm")
Connect-ExchangeOnline
$p=Get-SupervisoryReviewPolicyV2 -Identity $CCPolicyName

#Define a content search for mailboxes associated with the CC policy, with send dates after a specific date.
Connect-IPPSSession
New-ComplianceSearch $SearchName -ExchangeLocation $p.ReviewMailbox -AllowNotFoundExchangeLocationsEnabled $true -ContentMatchQuery $Query

#Start search and wait for completion
Start-ComplianceSearch -Identity $SearchName

Do{
Start-Sleep -s 30
$s=Get-ComplianceSearch -Identity $SearchName
(Get-Date -Format "HHmm") +" "+ $s.Status
}while($s.Status -ne "Completed")

#Start export search results and wait
New-ComplianceSearchAction -SearchName $SearchName -Export -ExchangeArchiveFormat SinglePst -Format FxStream

Do{
Start-Sleep -s 30
$a=Get-ComplianceSearchAction -identity ( $SearchName + '_Export')
(Get-Date -Format "HHmm") +" "+ $a.Status
}while($a.Status -ne "Completed")

#Launch a ClickOnce app via Edge
$a=Get-ComplianceSearchAction -identity ( $SearchName + '_Export') -includeCredential
Disconnect-ExchangeOnline -Confirm:$false

$temp=$a.Results.Substring(0,$a.Results.IndexOf(";"))
$url=[System.Uri]::EscapeDataString($temp.Substring($temp.IndexOf("https://")))

$token=$a.Results.Substring($a.Results.IndexOf("SAS token: ")).split(";")[0].split(" ")[2]
$ActionName=$a.Name.Replace(" ","+")

#Copy the export key to the clipboard
$token | clip

#If authentication is successful with an administrator account on Edge, ClickOnce content download will proceed. The app will launch.
#Paste the export key from the clipboard and download the .pst file locally.
start microsoft-edge:"https://complianceclientsdf.blob.core.windows.net/v16/Microsoft.Office.Client.Discovery.UnifiedExportTool.application?source=$url&name=$ActionName&trace=1&lite=1&customizepst=1"

````