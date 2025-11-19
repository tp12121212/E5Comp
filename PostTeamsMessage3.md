# Post to a Teams channel via the Microsoft Graph PowerShell SDK
This script is a sample that posts a message to a Teams channel via the Microsoft Graph PowerShell SDK.

## Preparation
Run the following two commands once in PowerShell with administrator privileges.
```
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
Install-Module Microsoft.Graph
```

## Example command to post channel chat to a Teams channel
```
$TeamName="Sales and Marketing"
$ChannelName="General"

# Function to post to a Teams channel
Function Post ($text){
$body= @"
{"body": {contentType: "text",content:"$text"}}
"@
New-MgTeamChannelMessage -ChannelId $Channel.Id -TeamId $Team.Id -BodyParameter $body
}

# Interactive logon
Connect-MgGraph -Scopes "Team.ReadBasic.All","Channel.ReadBasic.All"

$user=(Get-MgContext).Account

# Get the team from all teams the user is a member of
$Team=Get-MgUserJoinedTeam -UserId $user|?{$_.DisplayName -eq $TeamName}

#Get the channel from the team
$Channel=Get-MgAllTeamChannel -TeamId $Team.Id -filter "displayName eq '$ChannelName'"

$message="The current date is $(Get-date)."
Post $message
```