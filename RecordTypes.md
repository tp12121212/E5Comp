# About RecordType in Office 365 Audit Logs
[Docs](https://docs.microsoft.com/en-us/office/office-365-management-api/office-365-management-activity-api-schema#auditlogrecordtype) lists the possible values ​​for AuditRecordType, but the published list is not exhaustive and some values ​​may not be updated. The internally defined RecordTypes strings can be obtained from the error in the -RecordType argument of Search-UnifiedAuditLog. The enumeration value assigned to each RecordTypes string can be determined by specifying the RecordTypes as a string in the -RecordTypes argument of New-UnifiedAuditLogRetentionPolicy, then checking the RecordTypes value in the settings obtained by Set-UnifiedAuditLogRetentionPolicy. As of April 2022, the values ​​range from 1 to 171, with some missing numbers. For information on the log types automatically retained for 90 days with a valid license for Advanced Audit, see [Docs](https://docs.microsoft.com/ja-jp/microsoft-365/compliance/audit-log-retention-policies?view=o365-worldwide#more-information).

# List of Record Types in Office 365 Audit Logs
| Number | Name | Description in Docs | Retention for 1 year by default in Advanced Audit | Retention policy settings available from the UI | Sentinel support (tentative) |
|:---|:---------|:----:|:----:|:----:|:----:|
|1|ExchangeAdmin|Yes|Yes|Yes|Yes|Yes|
|2|ExchangeItem|Yes|Yes|Yes|
|3|ExchangeItemGroup|Yes|Yes|Yes|
|4|SharePoint|Yes|Yes|Yes|Yes
|5|SyntheticProbe||||
|6|SharePointFileOperation|Yes|Yes|Yes|Yes
|7|OneDrive|Yes|||
|8|AzureActiveDirectory|Yes|Yes|Yes|
|9|AzureActiveDirectoryAccountLogon|Yes|Yes||
|10|DataCenterSecurityCmdlet|Yes|||
|11|ComplianceDLPSharePoint|Yes|Yes||
|12|Sway|||Yes|
|13|ComplianceDLPExchange|Yes|Yes||
|14|SharePointSharingOperation|Yes|Yes |Yes|
|15|AzureActiveDirectoryStsLogon|Yes|Yes||
|16|SkypeForBusinessPSTNUsage|Yes|||
|17|SkypeForBusinessUsersBlocked|Yes|||
|18|SecurityComplianceCenterEOPCmdlet|Yes||Yes|
|19|ExchangeAggregatedOperation|Yes|Yes||
|20|PowerBIAudit|Yes||Yes|
|21|CRM|Yes| |Yes|
|22|Yammer|Yes||Yes|
|23|SkypeForBusinessCmdlets|Yes||Yes|
|24|Discovery|Yes||Yes|
|25|Microsoft Teams|Yes||Yes|Yes
|26|Missing number||||
|27|Missing number||||
|28|ThreatIntelligence|Yes|||
|29|MailSubmission|Yes|||
|30|MicrosoftFlow|Yes||Yes|
|31|AeD|Yes||Yes|
|32|Micros oftStream|Yes||Yes|
|33|ComplianceDLPSharePointClassification|Yes|Yes||
|34|ThreatFinder|Yes|||
|35|Project|Yes|Yes||
|36|SharePointListOperation|Yes|Yes|Yes|Yes
|37|SharePointCommentOperation|Yes|Yes||
|38|DataGovernance|Yes||Yes|
|39|Kaizala|Yes|||
|40| SecurityComplianceAlerts|Yes|||
|41|ThreatIntelligenceUrl|Yes|||
|42|SecurityComplianceInsights|Yes|||
|43|MIPLabel|Yes|||
|44|WorkplaceAnalytics|Yes||Yes|
|45|PowerAppsApp|Yes||Yes|
|46|PowerAppsPlan|Yes|||
|47|ThreatIntelligenceAtpContent|Yes|||
|48|Lab elContentExplorer|Yes||Yes|
|49|TeamsHealthcare|Yes|||
|50|ExchangeItemAggregated|Yes|Yes|Yes|Yes
|51|HygieneEvent|Yes|||
|52|DataInsightsRestApiAudit|Yes|||
|53|InformationBarrierPolicyApplication|Yes|Yes||
|54|SharePointListItemOperation|Yes|||
|55|SharePointContentTypeOperation|Yes|Yes|Yes|
|56|SharePointFieldOperation|Yes|Yes|Yes|
|57|MicrosoftTeamsAdmin|Yes||Yes||
|58|HRSignal|Yes|||
|59|MicrosoftTeamsDevice|Yes|||
|60|Microsoft Teams Analytics|Yes|||
|61|InformationWorkerProtection|Yes|||
|62|Campaign |Yes|Yes||
|63|DLPEndpoint|Yes||Yes|
|64|Air Investigation|Yes|||
|65|Quarantine|Yes||Yes|
|66|MicrosoftForms|Yes||Yes|
|67|ApplicationAudit|Yes|||
|68|ComplianceSupervisionExchange|Yes|Yes||Yes
|69|CustomerKeyServiceEncryption|Yes|Yes|Yes|
|70|OfficeNativ e|Yes|||
|71|MipAutoLabelSharePointItem|Yes|||
|72|MipAutoLabelSharePointPolicyLocation|Yes|||
|73|MicrosoftTeamsShifts|Yes|||
|74|SecureScore||||
|75|MipAutoLabelExchangeItem|Yes|||
|76|CortanaBriefing|Yes|||
|77|Search|Yes|||
|78|WDATPAlerts|Yes|||
|79| PowerPlatformAdminDlp||||
|80|PowerPlatformAdminEnvironment||||
|81|MDATPAudit|||Yes|
|82|SensitivityLabelPolicyMatch|Yes|||
|83|SensitivityLabelAction|Yes|||
|84|SensitivityLabeledFileAction|Yes|||
|85|AttackSim|Yes|||
|86|AirManualInvestigation|Yes|||
| 87|SecurityComplianceRBAC|Yes|||
|88|UserTraining|Yes|||
|89|AirAdminActionInvestigation|Yes|||
|90|MSTIC|Yes||Yes|
|91|PhysicalBadgingSignal|Yes|||
|92|TeamsEasyApprovals|||Yes|
|93|AipDiscover|Yes||Yes|
|94|AipSensitivityLabelAction|Yes||Yes|
|95|AipPro tectionAction|Yes||Yes|
|96|AipFileDeleted|Yes||Yes|
|97|AipHeartBeat|Yes||Yes|
|98|MCASAlerts|Yes|||
|99|OnPremisesFileShareScannerDlp|Yes||Yes|
|100|OnPremisesSharePointScannerDlp|Yes||Yes|
|101|ExchangeSearch|Yes||Yes|
|102|SharePointSearch|Yes||Yes|
| 103|PrivacyDataMinimization|Yes|||
|104|LabelAnalyticsAggregate||||
|105|MyAnalyticsSettings|Yes||Yes|
|106|SecurityComplianceUserChange|Yes|||
|107|ComplianceDLPExchangeClassification|Yes|||
|108|ComplianceDLPEndpoint||||
|109|MipExactDataMatch|Yes||Yes|
|110|MSDEResponseActions|||Yes|
|111|MSDEGeneralSettings|||Yes|
|112|MSDEIndicatorsSettings|||Yes|
|113|MS365DCustomDetection|Yes||Yes|
|114|MSDERolesSettings|||Yes|
|115|MAPGAlerts|||Yes|
|116|MAPGPolicy|||Yes|
|117|MAPGRemediation|||Yes|
|118|PrivacyRe mediationAction||||
|119|PrivacyDigestEmail||||
|120|MipAutoLabelSimulationProgress||||
|121|MipAutoLabelSimulationCompletion||||
|122|MipAutoLabelProgressFeedback||||
|123|DlpSensitiveInformationType|||Yes|
|124|MipAutoLabelSimulationStatistics||||
|125|LargeContentMetadata||||
|126|Microsoft365Group|||Yes|
|127|CDPMlInferencingResult||||
|128|FilteringMailMetadata||||
|129|CDPClassificationMailItem||||
|130|CDPClassificationDocument||||
|131|OfficeScriptsRunAction|||Yes||
|132|FilteringPostMai lDeliveryAction||||
|133|CDPUnifiedFeedback||||
|134|TenantAllowBlockList||||
|135|ConsumptionResource||||
|136|HealthcareSignal||||
|137|Missing number||||
|138|DlpImportResult|||Yes|
|139|CDPCompliancePolicyExecution
|140|MultiStageDisposition|||Yes|
|141|Priv acyDataMatch||||
|142|FilteringDocMetadata||||
|143|FilteringAttachmentInfo||||
|144|PowerBIDlp||||
|145|FilteringUrlInfo||||
|146|FilteringAttachmentInfo||||
|147|CoreReportingSettings|Yes|Yes||
|148|ComplianceConnector||||
|149|PowerPlatformLockbox ResourceAccessRequest|||Yes||
|150|PowerPlatformLockboxResourceCommand|||Yes||
|151|CDPPredictiveCodingLabel||||
|152|CDPCompliancePolicyUserFeedback||||
|153|WebpageActivityEndpoint||||
|154|OMEPortal|||Yes||
|155|CMImprovementActionChange||||
|156| FilteringUrlClick||||
|157| MipLabelAnalyticsAuditRecord||||
|158| FilteringEntityEvent||||
|159| FilteringRuleHits||||
|160| FilteringMailSubmission||||
|161| LabelExplorer||||
|162| Microsoft Managed Service Platform ||||
|163| PowerPlatformServiceActivity|||Yes||
|164| ScorePlatformGenericAuditRecord||||
|165| TimeTravelFilteringDocMetadata||||
|166| Alert||||
|167| AlertStatus||||
|168| AlertIncident||||
|169| IncidentStatus||||
|170| Case||||
|171| Case Investigation ||||
# About Audit Log Retention Policies
If you have a license that includes Advanced Audit, you can retain logs for up to one year. With an additional add-on license, you can set up to 10 years of log retention. However, as noted in [Docs](
https://docs.microsoft.com/ja-jp/microsoft-365/compliance/audit-log-retention-policies?view=o365-worldwide#create-and-manage-audit-log-retention-policies-in-powershell), only a limited number of log types can have retention policies set via the UI. To set retention policies for other logs, you must configure them separately using PowerShell.

## Example of Setting a Retention Policy via PowerShell
The following PowerShell command sets retention policies for log types 1 through 147 that are listed in Docs or can be set as targets in retention policies in the UI, without user input.
```
Connect-IPPSSession -UserPrincipalName xxxx@xxxx.onmicrosoft.com

$range=@();
$i=1;
$skip=@(5,26,27,74,79,80,104,108);
while ($i -le 109){if(!$skip.Contains($i)){$range+=$i;}$i++;}
$range+=113,147
New-UnifiedAuditLogRetentionPolicy -name "1Year Policy for All" -RetentionDuration TwelveMonths -priority 200 -recordtypes $range
```

# Regarding when the audit log retention policy takes effect
It appears that logs are retained according to the license status and retention policy at the time the logs are generated. Therefore, applying a retention policy now to currently remaining logs will not extend their retention period. However, even after the license expires, logs from the period the license was valid at the time will be retained according to the policy in effect at the time. Since there are only five default retention periods (90 days, 6 months, 9 months, 1 year, and 10 years), it can be assumed that logs are logically separated into containers according to the retention policy when they are generated, and old logs are truncated according to the validity period of each container.

# If you are unable to operate retention policies in the Compliance Center
If you have set a retention policy for a draft RecordType that is not published in PowerShell and the internal name of that RecordType has changed, you may not be able to view the list of retention policies or create new audit retention policies in the Compliance Center's [Audit Retention Policy](https://compliance.microsoft.com/auditlogsearch?viewid=Retention) screen. In such cases, you will need to delete the audit log retention policy that specifies the old RecordType name using PowerShell.
```
Connect-IPPSSession -UserPrincipalName xxxx@xxxx.onmicrosoft.com
#The deletion cmdlet will result in a pending deletion state.
Remove-UnifiedAuditLogRetentionPolicy -Identity "1Year Policy for All"
#You can immediately delete a policy by running the deletion cmdlet again with -ForceDeletion.
Remove-UnifiedAuditLogRetentionPolicy -Identity "1Year Policy for All" -ForceDeletion
```
The Get-UnifiedAuditLog command in PowerShell will also result in an error, so you must know the policy name before deleting it. If you have forgotten the policy name of an audit log you created in the past, refer to the page [here](https://github.com/YoshihiroIchinose/E5Comp/blob/main/Office365Audit.md) to extract the audit log of SecurityComplianceCenterEOPCmdlet, and look up the -Name argument in the Parameters of the log where Operation is New-UnifiedAuditLogRetentionPolicy.