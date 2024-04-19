## Definition of personal information for Japan
Defines patterns for Office 365 DLP that apply to personal information in Japan. At the moment we are defining an address and a phone number. The terminology is provided as an XML file as a custom sensitive information definition, so you can download it and use PowerShell to bring it into Office 365. Additional glossary definitions for Communication Compliance for Japan can be found on this page.

## Conducted once in advance
Launch PowerShell with administrative privileges and run:
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
Install-Module -Name ExchangeOnlineManagement
Ingesting XML as Custom Sensitive Information
Connect to Exchange Online from PowerShell
Import-Module ExchangeOnlineManagement
Connect-IPPSSession

## Upload a local XML file

New-DlpSensitiveInformationTypeRulePackage -FileData (Get-Content -Path "C:\Users\imiki\Desktop\Work\Comp\JPN_SIT.xml" -Encoding Byte)
If you have already uploaded and updated to a new version of the definition
Set-DlpSensitiveInformationTypeRulePackage -FileData (Get-Content -Path "C:\Users\imiki\Desktop\Work\Comp\JPN_SIT.xml" -Encoding Byte)


## Set up policies for DLP
Please use the following custom personal information definitions to set DLP policies for Office 365. If each match is found, it will be matched with a confidence level of 80.
SIT1.Address SIT2.Phone number SIT3.E-mail link


## SIT4.Japanese calendar

## SIT1.Address
The following regular expression matches a string consisting of prefecture + city + double-byte characters to detect the expression of an address in Japan.

## It detects patterns such as:
Konan, Minato-ku, Tokyo Chuo-ku,
Sapporo, Hokkaido

## regular expression
(北海道|東京都|(大阪|京都)府|(神奈川|和歌山|鹿児島)県|[^\x00-\x7F]{2}県)\s*[^\x00-\x7F]{1,6}[市郡区町村]
SIT2.Phone number

Detect the following phone number patterns:

2-digit area code 0x-xxxx-xxxx 3-digit area code 0xx-xxx-xxxx 4-digit area code 0xxx-xx-xxxx 5-digit area code 0xxxx-x-xxxx IP Phone 050-xxxx-xxxx



Mobile 070-xxxx-xxxx, 080-
xxxx-xxxx, 090-xxxx-xxxx

## Excluded patterns
Area code () (0x)-xxxx-xxxx toll-free 0120-xxx-xxx expressions including
country code +81-x-xxxx-xxxx

## Considerations for Office 365
1. Full-width alphanumeric characters are pre-converted to half-width alphanumeric characters.
When matching text in Office 365, full-width alphanumeric characters are converted to half-width alphanumeric characters and then matched with regular expressions.

2. Handling of half-width hyphens sandwiched between full-width
Note that half-width hyphens sandwiched between full-width hyphens, such as "full-width-full-width", are matched after a space is inserted before and after and replaced with " -".

## Phone Number Regular Expressions
Regular expressions for the phone number you are employing
(?<!\w|[\－ー―-])0(\d([\－ー―-]| - )\d{3}|\d{2}([\－ー―-]| - )\d{2}|\d{3}([\－ー―-]| - )\d|\d{4}([\－ー―-]| - )|[5789]0([\－ー―-]| - )\d{3})\d([\－ー―-]| - )\d{4}(?!\w|[\－ー―-])

## Commentary on the above
\d
Indicates a single single-digit number.

\d{4}
Indicates 4 single-byte digits.

([\－ー―-]| - )
It shows either full-width or half-width hyphens, full-width characters similar to hyphens, and "-" after conversion with Note 3 in mind. (Note: Since it is now necessary to escape full-width hyphens in the confidential information definition of Office 365, -about has been changed to be expressed with \-)

[5789]
Indicates one of the characters 5, 7, 8, or 9.

\in
Equivalent to [A-Za-z0-9], it shows one half-width alphanumeric character.

(?<!\w|[\－ー―-])
After the rest of the match is matched, it is a regular expression assertion that verifies that there are no preceded alphanumeric characters or hyphens, and it excludes similar strings such as serial codes such as E03-0000-0000 that partially contain the format of a phone number. Assertions require a fixed-length pattern, so we've reviewed the previous definition.

(?! \w|[\－ー―-])
After the rest of the body is matched, it is a regular expression assertion that verifies that there are no alphanumeric characters or hyphens trailing it, and it excludes similar strings such as serial codes such as 03-0000-0000-0000 that partially contain the format of a phone number. Assertions require a fixed-length pattern, so we've reviewed the previous definition.

 ## Types of hyphens
writing	UTF-8	Unicode Code Point	Description	Whether or not it is covered by this time
-	2D	U+002D	Hyphen-Minus	x
ー	E383BC	U+30FC	Katakana-Hiragana Prolonged Sound Mark	x
‐	E28090	Follow us - F t @	Another Hyphen	
‑	E28091	Follow us - F t @	Non-Breaking Hyphen	
‒	E28092	Follow us - F t @	Figure Dash	
–	E28093	Follow us - F t @	In Dash	
—	E28094	Follow us - F t @	In Dash	
―	E28095	Follow us - F t @	Horizontal Bar	x
−	E28892	Follow us - F t @	Minus Sign	
⁃	E28183	Follow us - F t @	Hyphen Bullet	
﹣	EFB9A3	2018 Compart AG	Small Hyphen-Minus	
ｰ	EFBDB0	2018 Compart AG	Halfwidth Katakana-Hiragana Prolonged Sound Mark	
－	EFBC8D	2018 Compart AG	Fullwidth Hyphen-Minus	x
SIT3. Email Link
Detects an email address that is a mailto link in an Office file instead of a string that is an email address.

## regular expression

mailto:[\w\-.!#$%&'*+\/=?^_`{|}~]+@[\w\-_]+\.[\w\-_.]+

## SIT4.Japanese Calendar

It detects the following patterns in the Japanese calendar:
December 12, the first year of Reiwa May
1, Showa 63 Pick-up
month Yoshi Day 2000 AD

## regular expression
(明治|大正|昭和|平成|令和|西暦)\s*([\d{1,4}|[元一二三四五六七八九十壱弐参拾〇○零]{1,4})\s*年\s*([\d{1,2}|[一二三四五六七八九十壱弐参拾〇○]{1,2})\s*月\s*([\d{1,2}|[元一二三四五六七八九十壱弐参拾〇○吉]{1,2})\s*日  

## References
Regular expression checker that allows you to check regular expressions on the web
Unidode character tool that allows you to check the code of a character