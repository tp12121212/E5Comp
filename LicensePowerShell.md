# Maintaining Security Groups Based on Service Plan Assignment
This is a sample PowerShell script that maintains security groups based on the assignment of service plans, which are application-level licenses.
This script checks the difference from the current group membership and adds internal users assigned the "INTUNE_A" service plan to a security group called "With_INTUNE_A." It also adds unassigned users to a security group called "Without_INTUNE_A." If a security group has not been created in advance, a new one will be created. You can also use a different service plan by replacing the $PlanName variable. For information on license SKUs and the service plans included in each SKU, see [here](https://docs.microsoft.com/en-us/azure/active-directory/enterprise-users/licensing-service-plan-reference). Please note that when deploying policies and features according to groups, some features, such as communication compliance, require you to use distribution groups rather than security groups.

# Verify connection, license (SKU), and service plan
```
$Id="xxxx@xxxx.onmicrosoft.com"
$Password = "xxxx"

# Generate credentials
$SecPass=ConvertTo-SecureString -String $Password -AsPlainText -Force
$Credential= New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $Id, $SecPass

Connect-AzureAD -Credential $Credential
$PlanName="INTUNE_A"

$skus=Get-AzureADSubscribedSku
$target=@()
$spid=""

# Obtain the corresponding SKU ID (there may be multiple) and Service Plan ID from the Service Plan name
foreach($sku in $skus){
foreach($sp in $sku.ServicePlans){
if($sp.ServicePlanName -eq $PlanName){
$target+=$sku.SkuId
$spid=$sp.ServicePlanId
}
}
}
```
## Maintaining internal member groups based on license ownership
```
#Target security group name
$LicencedGroup="With_$PlanName"

#Get licensed users (users who have the target SKU assigned and the corresponding service plan is not disabled)
$licensedUsers=@()
$users=Get-AzureADUser -All $true -Filter "userType eq 'Member'"
foreach($u in $users){
foreach($al in $u.AssignedLicenses){
if($target.Contains($al.SkuId) -and !$al.DisabledPlans.Contains($spid)){
$licensedUsers+=$u
break
}
}
}

#Maintain licensed user groups
$lg=Get-AzureADGroup -Filter "DisplayName eq '$LicencedGroup'"
if($lg -eq $null){
$lg=New-AzureADGroup -SecurityEnabled $true -MailEnabled $false -MailNickName $LicencedGroup -DisplayName $LicencedGroup
}

$mem=Get-AzureADGroupMember -All $true -ObjectId $lg.ObjectId
#Remove users who no longer have licenses from the group
foreach($m in $mem){
if(!$licensedUsers.Contains($m)){
Remove-AzureADGroupMember -ObjectId $lg.ObjectId -MemberId $m.ObjectId
}
}

#Add licensed users who are not registered in the group
foreach($lu in $licensedUsers){
if($mem -eq $null -or !$mem.Contains($lu)){
Add-AzureADGroupMember -ObjectId $lg.ObjectId -RefObjectId $lu.ObjectId
}
}
```
## Maintain a group of internal members who do not have licenses
```
#Target security group name
$UnLicencedGroup="Without_$PlanName"

#Get unlicensed users
$UsersWoL=@()
$users=Get-AzureADUser -All $true -Filter "userType eq 'Member'"
foreach($u in $users){
$found=$false
foreach($al in $u.AssignedLicenses){
if($target.Contains($al.SkuId) -and !$al.DisabledPlans.Contains($spid)){
$found=$true
break
}
}
if(!$found){$UsersWoL+=$u}
}
#Maintain unlicensed user groups
$ug=Get-AzureADGroup -Filter "DisplayName eq '$UnLicencedGroup'"
if($ug -eq $null){
$ug=New-AzureADGroup -SecurityEnabled $true -MailEnabled $false -MailNickName $UnLicencedGroup -DisplayName $UnLicencedGroup
}

$mem=Get-AzureADGroupMember -All $true -ObjectId $ug.ObjectId
#Remove licensed users from groups
foreach($m in $mem){
if(!$UsersWoL.Contains($m)){
Remove-AzureADGroupMember -ObjectId $ug.ObjectId -MemberId $m.ObjectId
}
}

#Add unlicensed users to groups
foreach($u in $UsersWoL){ 
if($mem -eq $null -or !$mem.Contains($u)){ 
Add-AzureADGroupMember -ObjectId $ug.ObjectId -RefObjectId $u.ObjectId 
}
}

````