# Use alert policies to issue alerts based on operations recorded in the audit log.
In the [Alert Policies](https://learn.microsoft.com/ja-jp/microsoft-365/compliance/alert-policies?view=o365-worldwide) section of the Purview Portal, you can check the conditions under which alerts are generated and administrators are notified, based on DLP settings and other factors.
You can also use these alert policies to set up alerts based on various operations recorded in the audit log.
However, when configuring via the [Web UI](https://compliance.microsoft.com/alertpoliciesv2), you cannot select specific audit log operations, making this somewhat less versatile.
On the other hand, you can use the [New-ProtectionAlert](https://learn.microsoft.com/ja-jp/powershell/module/exchange/new-protectionalert?view=exchange-ps) cmdlet provided in PowerShell to define alerts based on specific audit log operations.

## License
To set a threshold based on a specific time period or a 7-day anomaly, you need an MDO P2, E5, E5 Complinace, or eDiscovery and Audit license. The minimum time range is 60 minutes, and the minimum operation count threshold is 3. <br>
Even if you do not set a specific threshold, if you have a higher-level license, similar operations at 1-minute intervals will be aggregated into a single alert. If you do not have a higher-level license, similar operations at 15-minute intervals will be aggregated into a single alert. <br>
[Reference: Alert Aggregation](https://learn.microsoft.com/ja-jp/microsoft-365/compliance/alert-policies?view=o365-worldwide#alert-aggregation)

## Example of an alert based on sensitivity label operations

### An alert is generated each time a sensitivity label is deleted.
````
Connect-IPPSSession
New-ProtectionAlert -Category DataLossPrevention -Name SensitivityLabelRemoved -NotifyUser admin@xxxx.onmicrosoft.com -ThreatType Activity -Operation SensitivityLabelRemoved -AggregationType none -NotificationCulture ja-JP
````

### Generate an alert if the sensitivity label is changed more than three times within one hour.
````
Connect-IPPSSession
New-ProtectionAlert -Category DataLossPrevention -Name SensitivityLabelUpdated -NotifyUser admin@mxxx.onmicrosoft.com -ThreatType Activity -Operation SensitivityLabelUpdated -NotificationCulture ja-JP -AlertBy User -Threshold 3 -TimeWindow 60
````
### Alert Notification
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/alert/Alert01.png" height="400px">

### Alert Details
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/alert/Alert02.png" height="400px">

### Additional Information
Manual sensitivity label operations are recorded in the following RecodTyped and are applied to Office for the Office on SharePoint Online: Web, M365 Apps, and AIP Client operations are recorded. <br>
6 SharePointFileOperation
83 SensitivityLabelAction
94 AipSensitivityLabelAction
[Reference: About RecordTypes in Office 365 Audit Logs](https://github.com/YoshihiroIchinose/E5Comp/blob/main/RecordTypes.md)

The following common operations are recorded regardless of whether the RecordType is M365 Apps or AIP Client. Therefore, if you set an alert for the following operations, such as SensitivityLabelRemoved for the removal of a sensitivity label (as in the sample), or SensitivityLabelUpdated for the change of a sensitivity label, you can be alerted regardless of whether it is M365 Apps or AIP Client. <br>
SensitivityLabelApplied
SensitivityLabelUpdated
SensitivityLabelRemoved
SensitivityLabelPolicyMatched
SensitivityLabeledFileOpened

However, when operating sensitivity labels in Office for the Web, the operations are recorded with the following RecordType of SharePointFileOperation, which differs from those in M365 Apps and AIP Client. Therefore, if you want to target these operations, you must define a separate alert.
FileSensitivityLabelApplied
FileSensitivityLabelChanged
FileSensitivityLabelRemoved

### Limitations
When using alert policies to monitor sensitivity label operations, the following limitations apply:
1. While it is possible to set alerts for operations such as label changes and deletions, the logs are not separate and filtering based on the log content is not possible, so it is not possible to target only sensitivity label downgrade operations.
2. It is not possible to limit the users targeted by alerts by group. When setting an alert, a common threshold is set for all users in the tenant. (It may be possible to filter by user attributes, but at first glance, it does not appear possible to filter by Azure AD attributes or security groups.)
3. When an alert is issued and you check it, you can determine who performed a certain operation, when, and how many times. However, since the audit log does not contain the exact information recorded, you cannot determine what label was changed and how it was changed. <br>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/alert/Alert03.png"><br>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/alert/Alert04.png" height="400px"><br>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/alert/Alert05.png" height="400px"><br>

With these points in mind, if you want to investigate an alert, you can do so using the Activity Explorer UI or PowerShell. It is recommended to perform a detailed analysis based on the information extracted by [https://github.com/YoshihiroIchinose/E5Comp/blob/main/ActivityExplorerData.md]. Activity Explorer information not only includes the original and current sensitivity labels, but also identifies label event types such as LabelDowngraded and LabelUpgraded, which are not recorded in the audit log, allowing for more efficient filtering and investigation. <br>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/alert/Alert06.png" height="400px">

In the audit log, you can identify whether a sensitivity label was upgraded or downgraded by looking at the LabelEventType value of SensitivityLabelEventData.

| Value | Content | Meaning |
| :--- | :--- | :--- |
| 1 | LabelUpgraded | Changed to a label with a higher sensitivity level |
| 2 | LabelDowngraded | Changed to a label with a lower sensitivity level |
| 3 | LabelRemoved | Sensitivity label removed |
| 4 | LabelChangedSameOrder | Changed between sublabels with the same sensitivity level |