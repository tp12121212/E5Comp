# Delete specific logs in Microsoft Sentinel (Log Analytics)
Currently available as a preview feature, a data connector that allows you to ingest logs related to sensitivity label operations into Microsoft Sentinel is now available.
[Stream data from Microsoft Purview Information Protection to Microsoft Sentinel](https://learn.microsoft.com/ja-jp/azure/sentinel/connect-microsoft-purview)

This data connector allows you to ingest sensitivity label operation logs into Microsoft Sentinel with just a few clicks.
This makes it easy to create dashboards that analyze sensitivity label operations as a Power BI data source via Kusto queries, and to implement custom automated responses, such as notifying managers or relevant parties when a label is deleted or downgraded through Logic Apps. However, if you use the AIP UL Client, AIPHeartBeat logs are also ingested.
You may want to delete only these AIPHeartBeat logs. Therefore, on this page, we will share sample code that uses Azure Automation to delete only the AIPHeartBeat logs from Log Analytics.

## Preparation
1. Create an Azure Automation account with a system-assigned managed identity.
2. In Log Analytics, assign the "Data Purger" role to the system-assigned managed identity created above.
3. In your Azure Automation account, create a new runbook using PowerShell 5.1 and paste the following sample code.
4. Update the subscription ID, resource group name, and Log Analytics workspace name to reflect your environment.
5. Save and publish the runbook, then schedule it as needed.

## Notes
1. According to the [REST API Reference](https://learn.microsoft.com/ja-jp/rest/api/loganalytics/workspace-purge/purge?tabs=HTTP),
data erasure is not permitted without limit; it is limited to 50 requests per hour. It also supports only erasure operations required for GDPR compliance.
Requests unrelated to GDPR compliance may be rejected. Requests for deletion solely for log organization may be rejected.
2. After the deletion operation is initiated, the deletion will not occur immediately; it may take several hours for the process to be completed.
You can check the status via the following REST API:
[Get Purge Status](https://learn.microsoft.com/ja-jp/rest/api/loganalytics/workspace-purge/get-purge-status?tabs=HTTP)

## Azure Automation Sample Code
````
$subscription ="xxxxxx"
$resourcegroup="xxxxx"
$workspace="xxxxx"
$resource= "?resource=https://management.azure.com/"
$url = $env:IDENTITY_ENDPOINT + $resource
$headers = @{
"X-IDENTITY-HEADER" = $env:IDENTITY_HEADER
"Metadata"="True"
}
$accessToken = Invoke-RestMethod -Uri $url -Method 'GET' -Headers $headers

$uri="https://management.azure.com/subscriptions/$subscription/resourceGroups/$resourcegroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/purge?api-version=2020-08-01"
$headers = @{ 
"Content-Type" = "application/json" 
"Authorization"=("Bearer {0}" -f $accessToken.access_token)
}
$body = @"
{ 
"table": "MicrosoftPurviewInformationProtection", 
"filters":[{"column":"RecordType","operator":"==","value":97}]
}
"@
Invoke-WebRequest -Method "Post" -Uri $uri -Headers $headers -Body $body -UseBasicParsing
````