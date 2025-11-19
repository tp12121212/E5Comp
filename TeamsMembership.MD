# Overview
This script outputs a CSV file of any unauthorized Teams participation by monitored users belonging to a specific group, regardless of nesting. The ID used to connect must be able to access team member information. [Reference: Group Administrator Role](https://docs.microsoft.com/ja-jp/azure/active-directory/roles/permissions-reference#groups-administrator)

## Preparation
Run the following command once in PowerShell with administrator privileges.
```
Install-Module -Name AzureAD
Install-Module -Name MicrosoftTeams
```

## Sample script to check whether monitored users are participating in unauthorized Teams teams.
```
# Groups to audit (nesting not considered)
$GroupforAudit ="sg-Legal"
# Teams that members of the audited group are allowed to join.
$AllowedTeams=("Contoso Team", "Ask HR", "Operations")
# ID and password to use for connection
$Id="xxx@xxx.onmicrosoft.com"
$Password = "xxxxx"
$OutputFolder=[System.Environment]::GetFolderPath("Desktop")+"\"

#Generate Credential
$SecPass=ConvertTo-SecureString -String $Password -AsPlainText -Force
$Credential= New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $Id, $SecPass

Connect-AzureAD -credential $Credential

#Get monitored groups
$g=Get-AzureADGroup -SearchString $GroupforAudit

#Get details of allowed O365 groups
$AllowedTeamsGuid=@();
Foreach($t in $AllowedTeams){
$r=Get-AzureADGroup -SearchString $t
$AllowedTeamsGuid+=$r.ObjectId
}

#Get all Teams except allowed ones
Connect-MicrosoftTeams -credential $Credential
$Teams=Get-Team -NumberOfThreads 20
$TeamsGuid=@()
Foreach($tg in $Teams){
If(!$AllowedTeamsGuid.IndexOf($tg.GroupId) -ne -1)
{$TeamsGuid+=$tg.groupId.ToString()}
}

#Get target users
$UsersforAudit=Get-AzureADGroupMember -ObjectId $g.ObjectId -All $true|?{$_.UserType -eq "Member"}

$csv=@()
Foreach($u in $UsersforAudit){
#Get Teams membership for audited users
$membership=Get-AzureADUserMembership -ObjectId $u.ObjectID -All $true|?{$_.ObjectType -eq "Group"}|?{$TeamsGuid.IndexOf($_.ObjectId) -ne -1}

#Record unauthorized group membership
Foreach($m in $membership){
$line = New-Object PSObject | Select-Object UserName, UPN, TeamName, TeamGuid
$line.UserName=$u.DisplayName
$line.UPN=$u.UserPrincipalName
$line.TeamName=$m.DisplayName
$line.TeamGuid=$m.ObjectID
$csv+=$line
}
}

#CSV Output
$csv|Export-csv -Path ($OutputFolder+"UnAllowedMembership"+(Get-Date -Format "yyyyMMdd-HHmm")+".csv") -Encoding UTF8 -NoTypeInformation

```

## Sample Output
```
"UserName","UPN","TeamName","TeamGuid"
"Joni Sherman","JoniS@xxxx.OnMicrosoft.com","Sales and Marketing","8cd8e24a-71da-4891-b5ce-b38b7905944d"
"Joni Sherman","JoniS@xxxx.OnMicrosoft.com","Mark 8 Project Team","ac6d81e5-0f73-4127-9bde-228c4b9bc20f"
"Joni Sherman","JoniS@xxxx.OnMicrosoft.com","Retail","cc1265cc-842d-4dae-89d3-08caf2f5da77"
"Irvin Sayers","IrvinS@xxxx.OnMicrosoft.com","Digital Initiative Public Relations","e88fe2fc-9a3e-48f0-8a37-78a3db263866"
"Irvin Sayers","IrvinS@xxxx.OnMicrosoft.com","U.S. Sales","04a1f966-ff6f-4351-8418-23a28700528d"
"Grady Archie","GradyA@xxxx.OnMicrosoft.com","Contoso","09645b11-46ce-49bf-ad5d-53a9ac8feba5"
````