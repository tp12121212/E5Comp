# Posting to a Teams channel chat using the Graph API
This is a sample script that automates posting to a Teams channel chat, for use in CC/DLP testing, etc. This sample does not require administrator consent because it retrieves the team ID from the team (group) to which the account being used belongs, rather than referencing the entire team (group). A version that requires administrator consent is available here.

A sample that more easily posts to a Teams channel chat using the Microsoft Graph PowerShell SDK, rather than using the Graph API at a low level via GET/POST, is available here. ## Preparation
1. Register your application with Azure AD (https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/RegisteredApps)
2. Grant the following delegated permissions to the application's API: Team.ReadBasic.All, Channel.ReadBasic.All, and ChannelMessage.Send.
(ChannelMessage.Send cannot be executed by the application itself; it must be executed as a user delegate.)
3. Use the Client ID of your application to rewrite the parameter after the following URL:
https://login.microsoftonline.com/common/oauth2/authorize?response_type=code&client_id=XXXXX
Access this URL in a browser in InPrivate mode or similar, and authenticate with your account to consent to authorization.
If authentication is successful and you consent to authorization, you will receive the error message "AADSTS500113: No reply address is registered for the application." Once you reach this state, the application is OK.
4. Create a new Client Secret for the application you prepared in advance.
5. Use the following script to update the created application's Client ID, Client Secret, user ID/password, and the team name, channel name, and post content.
## Script body
Since the Graph API requires specification by ID, the script first identifies the team from the team name and the channel from the channel name before posting.
If you already know the team and channel IDs, you can skip steps "#Identify team ID" and "#Identify channel ID."

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
$url="https://graph.microsoft.com/v1.0/me/joinedTeams"
$r=(Invoke-RestMethod -Method GET -Uri $url -Headers $headers).value|?{$_.displayName -eq $TeamName}
if(!$r){"The group is not found";return}
$TeamID=$r.id

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