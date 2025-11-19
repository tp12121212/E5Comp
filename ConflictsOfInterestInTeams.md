# Overview
This script is a way to monitor communication between groups with conflicting interests. It lists, in CSV format, members of a specific Group A and a specific Group B who coexist as members of an unauthorized Teams team, regardless of nesting. The ID used for Connect must be able to access team member information. [Reference: Group Administrator Role](https://docs.microsoft.com/ja-jp/azure/active-directory/roles/permissions-reference#groups-administrator)

## Preparation
Execute the following once in PowerShell with administrator privileges.
```
Install-Module -Name AzureAD
Install-Module -Name MicrosoftTeams
```

## Sample script to check for Teams teams in which members of Group A and Group B, who have conflicting interests, share membership.
```
# Groups to audit (nesting not considered)
$GroupforAuditA ="sg-Legal"
$GroupforAuditB ="sg-Finance"
# Teams where members of Group A and Group B are allowed to coexist
$AllowedTeams=("Contoso Team", "Ask HR", "Operations")
# ID and password to use for connection
$Id="xxx@xxx.onmicrosoft.com"
$Password = "xxxxx"
$OutputFolder=[System.Environment]::GetFolderPath("Desktop")+"\"

#Generate Credential
$SecPass=ConvertTo-SecureString -String $Password -AsPlainText -Force
$Credential= New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $Id, $SecPass

Connect-AzureAD -credential $Credential
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

#Get monitored groups
$ga=Get-AzureADGroup -SearchString $GroupforAuditA
$gb=Get-AzureADGroup -SearchString $GroupforAuditB

#Get users in Group A
$UsersforAuditA=Get-AzureADGroupMember -ObjectId $ga.ObjectId -All $true|?{$_.UserType -eq "Member"}

#Get users in Group B not included in Group A
$UsersforAuditB=Get-AzureADGroupMember -ObjectId $gb.ObjectId -All $true|?{$_.UserType -eq "Member"}|?{$UsersforAuditA.IndexOf($_) -eq -1}

#Get all memberships for Group B members in advance
$UserBTable=@()
$MembershipBTable=@()
Foreach($ub in $UsersforAuditB){
$membershipB=Get-AzureADUserMembership -ObjectId $ub.ObjectID -All $true |?{$_.ObjectType -eq "Group"} |?{$TeamsGuid.IndexOf($_.ObjectId) -ne -1}
$UserBTable+=$ub
#Add a comma to create a two-dimensional array
$MembershipBTable+=,$membershipB
}

$csv=@()
#Find Teams teams that both members of Group A and Group B belong to
Foreach($ua in $UsersforAuditA){
#Get memberships for members of Group A 
$membershipA=Get-AzureADUserMembership -ObjectId $ua.ObjectID -All $true |?{$_.ObjectType -eq "Group"}|?{$TeamsGuid.IndexOf($_.ObjectId) -ne -1} 
Foreach($ma in $membershipA){ 
Foreach($mb in $MembershipBTable){ 
$i=$mb.IndexOf($ma) 
If($i -ne -1){ 
$ub=$UserBTable[$MembershipBTable.IndexOf($mb)] 
$line = New-Object PSObject | Select-Object UserAName, UserAUPN, UserBName, UserBUPN, TeamName, TeamGuid 
$line.UserAName=$ua.DisplayName 
$line.UserAUPN=$ua.UserPrincipalName 
$line.UserBName=$ub.DisplayName 
$line.UserBUPN=$ub.UserPrincipalName $line.TeamName=$mb[$i].DisplayName
$line.TeamGuid=$mb[$i].ObjectID
$csv+=$line
}
}
}
}

#CSV Output
$csv|Export-csv -Path ($OutputFolder+"ConflictMembership"+(Get-Date -Format "yyyyMMdd-HHmm")+".csv") -Encoding UTF8 -NoTypeInformation
```

## Sample Output
```
"UserAName","UserAUPN","UserBName","UserBUPN","TeamName","TeamGuid"
"Joni Sherman","JoniS@xxxx.OnMicrosoft.com","Debra Berger","DebraB@xxxx.OnMicrosoft.com","Sales and Marketing","8cd8e24a-71da-4891-b5ce-b38b7905944d"
"Joni Sherman","JoniS@xxxx.OnMicrosoft.com","Pradeep Gupta","PradeepG@xxxx.OnMicrosoft.com","Sales and Marketing","8cd8e24a-71da-4891-b5ce-b38b7905944d"
"Grady Archie","GradyA@xxxx.OnMicrosoft.com","Pradeep Gupta","PradeepG@xxxx.OnMicrosoft.com","Mark 8 Project Team","ac6d81e5-0f73-4127-9bde-228c4b9bc20f"
"Grady Archie","GradyA@xxxx.OnMicrosoft.com","Megan Bowen","MeganB@xxxx.OnMicrosoft.com","Mark 8 Project Team","ac6d81e5-0f73-4127-9bde-228c4b9bc20f"
"Grady Archie","GradyA@xxxx.OnMicrosoft.com","Megan Bowen","MeganB@xxxx.OnMicrosoft.com","Contoso","09645b11-46ce-49bf-ad5d-53a9ac8feba5"
"Grady Archie","GradyA@xxxx.OnMicrosoft.com","Diego Siciliani","DiegoS@xxxx.OnMicrosoft.com","Contoso","09645b11-46ce-49bf-ad5d-53a9ac8feba5"
````