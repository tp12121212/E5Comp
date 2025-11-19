# Scripts related to Insider Risk Management's HR Connector
## Importing CSV files uploaded to Azure Storage into IRM's HR Connector via an Azure Automation script
The first sample is a script that imports CSV files uploaded to Azure Storage into IRM's HR Connector via an Azure Automatino script. By periodically uploading CSV files from HR to Azure Storage, you can schedule the import of HR data into IRM using Azure Automation without requiring a continuously running on-premises server.

### Preparation
1. Create a storage account in Azure and create a file share named "hrdata" there.
2. Upload the CSV file to be imported by the IRM HR Connector to the file share named "HR_Resignation.txt" in UTF-8 with BOM format.
3. Paste the script published here into Notepad or another application and save it as "HRConnector.ps1" in UTF-8 with BOM format.
4. Upload the above file to the "hrdata" file share named in 1.
6. Create an Azure Automation account.
7. In your Azure Automation account, save the connection key to the file share named "storageKey" as an encrypted string variable.
8. In your Azure Automation account, enter the Client Secret of the application registered in Azure AD for the IRM Connector. Save the encrypted string as a variable named "appSecret."
9. In your Azure Automation account, create a runbook in PowerShell format with runtime version 5.1.
10. Paste the following script into the above runbook.
11. Update the $tenantId, $appId, and $jobId in the script to match your environment and publish the script.
12. Verify the runbook's operation and confirm that the script successfully imported the CSV data.
13. Schedule the runbook to run based on the frequency with which the CSV files imported from HR are uploaded to Azure Storage.

### Script body to register with Azure Automation
````
$StorageConnection = Get-AutomationVariable -Name 'storageKey'
$strctx = New-AzureStorageContext -ConnectionString $StorageConnection
$appSecret = Get-AutomationVariable -Name 'appSecret'
$tenantId="xxxxx" #Azure AD tenant ID
$appId="xxxxx" #ID of the application registered for IRM in Azure AD
$jobId="xxxx" #ID specified in the HR Connector settings for IRM

$path=Get-Location
Get-AzureStorageFileContent -ShareName "hrdata" -Path "HR_Resignation.txt" -Context $strctx
Get-AzureStorageFileContent -ShareName "hrdata" -Path "HRConnector.ps1" -Context $strctx
.\HRConnector.ps1 -tenantId $tenantId -appId $appId `
-appSecret $appSecret -jobId $jobId -filePath "$path\HR_Resignation.txt"
````

## Create a mail-enabled security group and maintain group membership based on a CSV file imported by the HR Connector.
The second sample creates a mail-enabled security group named "IRMTargetGroup" and adds users listed in a CSV file from HR to "IRMTargetGroup." If the number of users increases or decreases in the CSV file from HR, the members of "IRMTargetGroup" will also change accordingly. By creating this "IRMTargetGroup" group, you can limit the scope of IRM policies. Using the same method, you can also maintain specific groups based on CSV files on Azure Storage, not just for IRM purposes.

### Prerequisites
#### If you have not yet completed the previous steps
1. Create a storage account in Azure and create a file share named "hrdata" there.
2. Upload the CSV file to be imported by the IRM HR Connector to the file share with the file name "HR_Resignation.txt" in UTF-8 with BOM format.
3. Create an Azure Automation account.
4. In the Azure Automation account, save the connection key to the file share from step 1 as an encrypted string variable named "storageKey."
#### New Steps
5. In the Azure Automation account, import the "ExchangeOnlineManagement" module from the Gallery.
(Runtime version is 5.1)
6. In the Azure Automation account, register the ID and password of an account with Exchange administrator privileges named "Office 365" for Exchange Online connectivity.
7. In the Azure Automation account, create a runbook in PowerShell format with runtime version 5.1.
8. Paste the following script into the runbook and publish it.
9. Verify that the runbook works and that an email-enabled security group called "IRMTargetGroup" is created with the members listed in the CSV file.
10. Schedule the runbook to run according to the frequency with which the CSV file from HR is uploaded to Azure Storage.

### Script body for registering with Azure Automation
````
# Get the connection string for the storage share
$StorageConnection = Get-AutomationVariable -Name 'storageKey'
$strctx = New-AzureStorageContext -ConnectionString $StorageConnection

# Get the credential for the Azure AD connection
$Credential = Get-AutomationPSCredential -Name "Office 365"

# Get the HR CSV file
$path=Get-Location
Get-AzureStorageFileContent -ShareName "hrdata" -Path "HR_Resignation.txt" -Context $strctx
$csv=Import-Csv -Path "$path\HR_Resignation.txt"

# Create a list of users in HR CSV
$TargetUsers=@()
foreach($line in $csv){ 
$TargetUsers+=$line.EmailAddress.ToLower()
}

# Show current users in CSV
"Current users in CSV ("+$TargetUsers.count+" users)"
$TargetUsers

# Connect Azure AD
Connect-ExchangeOnline -Credential $Credential

# Get "IRMTargetGroup", if it doesn't exist, create it
$IRMTargetGroup="IRMTargetGroup"
$g=Get-DistributionGroup -Identity $IRMTargetGroup -ErrorAction Ignore
$members=@()
if($g -eq $null){ 
$g=New-DistributionGroup -Name $IRMTargetGroup -Type "Security" 
"IRMTargetGroup is newly created."
}
else{ 
"IRMTargetGroup is found." 

#Get current users in "IRMTargetGroup" 
$mem=Get-DistributionGroupMember -Identity $IRMTargetGroup -ResultSize 2000 

#Create a list of users in "IRMTargetGroup" 
foreach($m in $mem){ 
if($m.PrimarySmtpAddress) 
{$members+=$m.PrimarySmtpAddress.ToLower()} 
else{$members+=($m.Identity+"@"+$m.OrganizationalUnitRoot).ToLower()} 
} 

#Show current users in IRMTargetGroup 
"Current users in IRMTargetGroup ("+$members.count+" users)" 
$members
}

# Remover users from "IRMTargetGroup" if they are not in HR CSV
foreach($m in $members){ 
if(!$TargetUsers.Contains($m)){ Remove-DistributionGroupMember -Identity $IRMTargetGroup -Member $m -Confirm:$false 
"$m is removed." 
}
}

# Add users to "IRMTargetGroup" if they are not in the group but are in HR CSV
foreach($u in $TargetUsers){ 
if(!$members.Contains($u)){ 
Add-DistributionGroupMember -Identity $IRMTargetGroup -Member $u 
"$u is added." 
}
}
Disconnect-ExchangeOnline -Confirm:$false
"Done"
````