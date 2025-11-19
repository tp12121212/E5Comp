# Posting to a Teams channel chat using the Graph API
This is a sample script that automates posting to a Teams channel chat, for use in CC/DLP testing, etc. A version that does not require administrator consent is available here.

## Preparation
1. Register an application with Azure AD (https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/RegisteredApps)
2. Grant the following delegated permissions to the created application's API: Group.Read.All, Channel.ReadBasic.All, and ChannelMessage.Send. Obtain administrator consent.
(ChannelMessage.Send cannot be executed by the application itself; it must be executed as a user delegate, while Group.Read.All requires administrator consent.)
3. Create a new Client Secret for the application you created.
4. Use the following script to update the created application's Client ID, Client Secret, user ID/password, and the destination team name, channel name, and post content.
## Script Body
Since the Graph API requires ID specification, the process first identifies the team from the team name and the channel from the channel name before posting.
If you already know your team or channel IDs, you can skip "#Identify team ID" and "#Identify channel ID".

````
$clientID = "xxxx"
$clientSecret="xxxxx"
$Id="xxxx@xxxx.onmicrosoft.com"
$Password = "xxxx"

$TeamName="Sales and Marketing"
$ChannelName="General" #Even if "General" is displayed in Japanese, "General" must be specified.
$Text="It's a nice day today."

#Get a token
$tokenBody = @{
Grant_Type = "password"
client_Id = $clientId
client_secret = $clientSecret
username = $Id
password = $Password
resource = "https://graph.microsoft.com"
}

$tokenResponse = Invoke-RestMethod "https://login.microsoftonline.com/common/oauth2/token" -Method Post -Body $tokenBody -ErrorAction STOP

$headers = @{
"Authorization" = "Bearer $($tokenResponse.access_token)"
"Content-type" = "application/json"
}

#Identify team ID
$url="https://graph.microsoft.com/v1.0/groups?`$filter=displayName+eq+'$TeamName'"
$r=Invoke-RestMethod -Method GET -Uri $url -Headers $headers
if(!$r.value){"The group is not found";return}
$TeamID=$r.value[0].id

#Identify channel ID
$url="https://graph.microsoft.com/v1.0/teams/$TeamID/channels?`$filter=displayName+eq+'$ChannelName'"
$r=Invoke-RestMethod -Method GET -Uri $url -Headers $headers
if(!$r.value){"The channel is not found";return}
$ChannelID=$r.value[0].id

#Posting Destination
$url="https://graph.microsoft.com/v1.0/teams/$TeamID/channels/$ChannelID/messages"

#Posting Content
$body= @"
{"body": {contentType: "text",content:"$Text"}}
"@

#Posting When posting in Japanese, you must specify -ContentType "application/json; charset=utf-8"
Invoke-RestMethod -Method POST -Uri $url -Body $body -Headers $headers -ContentType "application/json; charset=utf-8"
````