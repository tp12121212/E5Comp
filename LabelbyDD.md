# Performing sensitivity labeling operations via drag-and-drop on the desktop
This script is a sample that uses the labeling and label removal PowerShell cmdlets included in the Purview Information Protection Client
to apply sensitivity labels to multiple files or remove labels via drag-and-drop.
## Overview
Since the PowerShell script itself does not accept drag-and-drop operations,
you will need to create a shortcut that calls the PowerShell script, and drag and drop the files using that shortcut.
Note that the labeling script requires you to specify the GUID of the label you want to apply,
and the label removal script requires you to specify a reason for the label, depending on how your administrator configured the label.

## Preparation
1. Install the Purview Information Protection Client from [here](https://aka.ms/aipclient)
2. Launch PowerShell with administrative privileges and run the following once beforehand.
```
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
```
3. In PowerShell, use Get-FileStatus to specify the path to the file to which you want to automatically assign a sensitivity label, and obtain the GUID of the sensitivity label.
```
Get-FileStatus "file path"
```
The SubLabelId returned by running the above command will be the GUID of the target sensitivity label.
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/LabelStatus.png">
## Bulk Sensitivity Labeling Setup Procedure
1. Paste the following script into Notepad, modify it to specify the GUID of the sensitivity label obtained in Preparation 3, and run Labeling.ps1. Save as
```
$label="label GUID"

$Args | foreach{
$item = Get-Item -LiteralPath $_
$item.FullName
$status=get-filestatus $item
if($status.SubLabelId -ne $label){
$status=Set-FileLabel -path $item -Labelid $label
"The labeling operation was " +$status.Status
}else{
"The label has already been set."
}

}
```
2. Create a shortcut that calls the script from step 1.
Specify the location of the script by specifying the following as the shortcut's destination:
```
powershell.exe -NoProfile -ExecutionPolicy RemoteSigned -noexit -File "C:\Users\MeganBowen\Desktop\Labeling.ps1"
```
3. Drag and drop the files you want to label onto the shortcut above to label them all at once.

## Setup for bulk sensitivity label removal
1. Paste the following script into Notepad, edit the $Justification section appropriately, and save it as RemovingLabel.ps1.
```
$Justification="For sharing with external customers"

$Args | foreach{
$item = Get-Item -LiteralPath $_
$item.FullName
$status=get-filestatus $item
if($status.MainLabelName -eq $null){
"No label"
exit
}
"Current label:" + $status.MainLabelName +" / "+ $status.SubLabelName
if($status.SubLabelId -ne $null){
$status=Remove-FileLabel -path $item -JustificationMessage $Justification
"The operation of label removal was " + $status.Status
}
}
```
2. Create a shortcut that calls the script from step 1.
Specify the following as the shortcut's destination and specify the location of the script.
```
powershell.exe -NoProfile -ExecutionPolicy RemoteSigned -noexit -File RemovingLabel.ps1
```
3. Drag and drop the files you want to label onto the shortcut to remove labels in bulk.

## Other Considerations
### Permissions
This script uses user permissions. Therefore, if the user does not have full control permissions, the label of already protected files cannot be changed.
Administrators and others who wish to recover files must configure a [SuperUser](https://learn.microsoft.com/ja-jp/azure/information-protection/configure-super-users) and use that user account.
### Usage
When using sensitivity labels in a company, making users aware of the sensitivity of each file can be important in meeting confidentiality management requirements for trade secrets. To ensure label protection remains a mere formality, carefully consider whether or not to use such script-based bulk processing.
### Script Deployment
To deploy a PowerShell script and make it available on other devices, each user must either right-click the script, view its properties, and allow it, or sign the script with a certificate from a trusted certificate authority to detect tampering.