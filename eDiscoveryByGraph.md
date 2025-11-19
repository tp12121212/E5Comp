# This script is a sample of searching eDiscovery (Premium) mailboxes using the Graph API.
This script demonstrates how to use the Graph API to automate the following steps: create an eDiscovery (Premium) case, add a custodian (the user being investigated), specify the custodian's mailbox as the search target, register search keywords to find credit card number SITs from emails, create a review set, and update the review set with the search results. If you want to reuse an existing case or review set, use the commented-out code to retrieve existing objects.
The custodian addition section uses an array, allowing it to be expanded to multiple investigations.
The search keywords section searches using sensitive information types, which are now available in EXO. Here, the credit card number SIT is specified by its GUID.
To specify a custom SIT, open the SIT from the Purview portal; replace the SIT's GUID in the URL.

## Preparation
Open PowerShell with administrative privileges and run the following once. Some parts only work in the Beta version, so be sure to also install the Beta version.
```
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
Install-Module Microsoft.Graph
Install-Module Microsoft.Graph.Beta.Security
```

## Script
```
# Variables
$CaseName="Case 1 by Graph"
$SearchName="Search 1 by Graph"
$ReviewsetName="Review Set 1"
$emails=@("alexw@xxxx.onmicrosoft.com","admin@xxxx.onmicrosoft.com")
$query='SensitiveType="50842eb7-edc8-4019-85dd-5a5c1f2bb085|1..500|1..100" AND sent>=2025-01-01 AND sent<=2025-03-31 AND kind:email'

#1. Connect with Appropriate Permissions
Connect-MgGraph -Scopes "eDiscovery.ReadWrite.All"

#2-1. Create a Case
$case=New-MgSecurityCaseEdiscoveryCase -DisplayName $CaseName

#2-2. Get an Existing Case
#$case=Get-MgSecurityCaseEdiscoveryCase -Filter "DisplayName eq '$CaseName'"

#3-1. Add Custodians
$custodians=@()
Foreach($email in $emails){
$custodians+=New-MgSecurityCaseEdiscoveryCaseCustodian -EdiscoveryCaseId $case.id -Email $email
}

#3-2. Get an Existing Custodian
#$custodians=Get-MgSecurityCaseEdiscoveryCaseCustodian -EdiscoveryCaseId $case.id

#4-1. Add Mailbox
$mailboxes=@()
Foreach($cust in $custodians){
$mb=New-MgSecurityCaseEdiscoveryCaseCustodianUserSource -EdiscoveryCaseId $case.id -EdiscoveryCustodianId $cust.Id -Email $cust.Email -IncludedSources "mailbox"
$baseURL="https://graph.microsoft.com/v1.0/security/cases/ediscoveryCases/"+$case.id+"/custodians/"
$mailboxes+=$baseURL+$cust.id+"/userSources/"+$mb.Id
}

#4-2. Get Existing Target Mailboxes
#$mailboxes=@()
#Foreach($cust in $custodians){
# $mb=get-MgSecurityCaseEdiscoveryCaseCustodianUserSource -EdiscoveryCaseId $case.id -EdiscoveryCustodianId $cust.Id
# $baseURL="https://graph.microsoft.com/v1.0/security/cases/ediscoveryCases/"+$case.id+"/custodians/"
# $mailboxes+=$baseURL+$cust.id+"/userSources/"+$mb.Id
#}

#5-1. Search registration
$params = @{ 
displayName = $SearchName 
contentQuery = $query 
"custodianSources@odata.bind"=$mailboxes
}
$search=New-MgSecurityCaseEdiscoveryCaseSearch -EdiscoveryCaseId $case.id -BodyParameter $params

#5-2. Get existing search
#$search=Get-MgSecurityCaseEdiscoveryCaseSearch -EdiscoveryCaseId $case.id|?{$_.DisplayName -eq $searchName}

#6-1. Create a review set
$reviewset=New-MgSecurityCaseEdiscoveryCaseReviewSet -EdiscoveryCaseId $case.id -DisplayName $ReviewsetName

#6-2. Get an existing review set
#$reviewset=MgSecurityCaseEdiscoveryCaseReviewSet -EdiscoveryCaseId $case.id -Filter "displayName eq '$ReviewsetName'"

#7-1. Add search results to a review set
$params = @{
search = @{
id = $search.id
}
additionalDataOptions = "linkedFiles"
}
#This section will result in an error if you are not using the Beta Graph API.
Add-MgBetaSecurityCaseEdiscoveryCaseReviewSetToReviewSet -EdiscoveryCaseId $case.id -EdiscoveryReviewSetId $reviewset.id -BodyParameter $params
````