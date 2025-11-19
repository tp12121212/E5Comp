# Alerting on Sensitivity Label Downgrade Operations in Sentinel
This is a sample Kusto query that uses logs of sensitivity label operations imported with the Microsoft Purview Information Protection data connector to create an alert for multiple sensitivity label downgrades within a certain period of time.

## Base Kusto Query
The following Kusto query narrows down logs from the Microsoft Purview Information Protection table to those with a LabelEventType of "LabelDowngraded." Since we want to alert only on multiple label downgrades within a certain period of time, we round the logs to an hourly range and aggregate them. In this example, we'll alert on three or more operations per hour.

However, a limitation of this simple aggregation method is that operations that cross the hour are not aggregated. For example, an operation occurring once between 2:00 PM and two between 3:00 PM would not trigger an alert.
Furthermore, if you simply aggregate using summarize, detailed information about each log will be lost, so before aggregating, the file name, previous sensitivity label, new sensitivity label,
the reason for the label change, and the time of the operation are collected into a property bag using bag_pack, and then concatenated as a JSON array using make_set when aggregating.
```
MicrosoftPurviewInformationProtection
| where LabelEventType == "LabelDowngraded"
| extend LabelDetail = bag_pack("File",ObjectId,"OldLabel",OldSensitivityLabelId,"NewLabel",SensitivityLabelId,"Justification",JustificationText, "Time",TimeGenerated)
| summarize counts=count(), LabelDetails=make_set(LabelDetail) by UserId, length=bin(TimeGenerated, 60m)
| where counts >=3
```

## Use GUIDs for Sensitivity Labels as Display Names
Since each sensitivity label is recorded as a GUID in the MicrosoftPurviewInformationProtection log, if you want to view sensitivity labels by their display names,
you need to prepare a mapping table that converts GUIDs to display names. This is also explained on the official website below.
[Mapping Sensitivity Label Display Names](https://learn.microsoft.com/en-us/azure/sentinel/connect-microsoft-purview#known-issues-and-limitations)

## Obtaining Sensitivity Label Information with PowerShell
To create the mapping table described on the above site, use Exchange Online PowerShell to reference the information of predefined sensitivity labels. The following sample uses PowerShell to output the GUID and display name information of sensitivity labels in JSON format, which is easy to paste into a Kusto query.
```
Connect-IPPSSession
$labels=Get-Label|?{$_.ContentType.IndexOf("File") -ne -1}
$a=@{}
$labels|%{$a.add($_.Guid.ToString(),$_.Name)}
$a|convertto-json
```
## The above PowerShell command will produce the following JSON-formatted output.
```
{
"30b5e379-19c1-4793-8850-770933e0bf5e": "Non-Business",
"c7861feb-52bc-4794-9c51-6e7f088ee93c": "Open Encryption",
"2605be26-f499-4a91-993d-520e8650d0c6": "Confidential",
"5759c9fa-cc5c-4c01-9858-a0e53b65e13e": "Business"
}
```
## Example Kusto query replacing sensitivity labels with display names
By pasting the output from the previous PowerShell command and replacing the GUIDs with display names, you can view sensitivity labels by display name in Sentinel alerts. When pasting a JSON Kusto query, please note that you must enclose each GUID and display name mapping line in single quotation marks ('), and that you must follow the JSON definition with a semicolon (;) and no blank lines before entering the query.
```
let labelsMap = parse_json('{'
'"30b5e379-19c1-4793-8850-770933e0bf5e": "Non-Business",'
'"c7861feb-52bc-4794-9c51-6e7f088ee93c": "Open Encryption",'
'"2605be26-f499-4a91-993d-520e8650d0c6": "Confidential",'
'"5759c9fa-cc5c-4c01-9858-a0e53b65e13e": "Business"'
'}');
MicrosoftPurviewInformationProtection
| where LabelEventType == "LabelDowngraded"
| extend SensitivityLabelName = iif(isnotempty(SensitivityLabelId), tostring(labelsMap[tostring(SensitivityLabelId)]), "")
| extend OldSensitivityLabelName = iif(isnotempty(OldSensitivityLabelId), tostring(labelsMap[tostring(OldSensitivityLabelId)]), "")
| extend LabelDetail = bag_pack("File",ObjectId,"OldLabel",OldSensitivityLabelName,"NewLabel",SensitivityLabelName,"Justification",JustificationText, "Time",TimeGenerated)
| summarize counts=count(), LabelDetails=make_set(LabelDetail) by UserId, length=bin(TimeGenerated, 60m)
| where counts >=3
```
## Kusto Query Sample Results
The above query returns 3 records every hour. The query results will show which users have had their sensitivity labels downgraded more than once and how many times. The specific operations can be viewed in LabelDetails.
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/SensitiveLabelKusto.png"/>