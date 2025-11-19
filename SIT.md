# Definition of Personal Information for Japan
This defines patterns that qualify as personal information in Japan for Office 365 DLP. Currently, addresses and phone numbers are defined. The terms are provided as custom sensitive information definitions in the XML file [XML](https://github.com/YoshihiroIchinose/E5Comp/blob/main/SIT/JPN_SIT.xml). Please download this file and import it into Office 365 using PowerShell. Other Communication Compliance glossary definitions for Japan are available on this page [XML](https://github.com/YoshihiroIchinose/JPN-CC/blob/master/README.md).

# Run this once in advance
## Launch PowerShell with administrative privileges and run the following:
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
Install-Module -Name ExchangeOnlineManagement

# Import XML as custom sensitive information
## Connect to Exchange Online via PowerShell
Import-Module ExchangeOnlineManagement
Connect-IPPSSession

## Upload a local XML file
New-DlpSensitiveInformationTypeRulePackage -FileData (Get-Content -Path "C:\Users\imiki\Desktop\Work\Comp\JPN_SIT.xml" -Encoding Byte)

## If you have already uploaded the definition and are updating it to a new version
Set-DlpSensitiveInformationTypeRulePackage -FileData (Get-Content -Path "C:\Users\imiki\Desktop\Work\Comp\JPN_SIT.xml" -Encoding Byte)

# Set DLP policy
Use the following imported custom personal information definition to protect Office Please set up your 365 DLP policies, etc. If a match is found, it will be matched with a confidence level of 80.
SIT1. Address
SIT2. Phone Number
SIT3. Email Link
SIT4. Japanese Calendar
# SIT1. Address
The following regular expression detects Japanese address expressions by matching a string consisting of prefecture + city/ward/town/village + full-width characters.
## Detects the following patterns:
Konan, Minato Ward, Tokyo
Chuo Ward, Sapporo City, Hokkaido

## Regular Expression
(Hokkaido | Tokyo | (Osaka | Kyoto) Prefecture | (Kanagawa | Wakayama | Kagoshima) Prefecture | [^\x00-\x7F]{2} Prefecture)\s*[^\x00-\x7F]{1,6}[City/County/Ward/Town/Village]

# SIT2. Phone Number
## Detects the following phone number pattern:
2-digit area code: 0x-xxxx-xxxx
3-digit area code: 0xx-xxx-xxxx
4-digit area code: 0xxx-xx-xxxx
5-digit area code: 0xxxx-x-xxxx
IP phone: 050-xxxx-xxxx
Mobile phone: 070-xxxx-xxxx, 080-xxxx-xxxx, 090-xxxx-xxxx
## Excluded patterns
Area code: () (0x)-xxxx-xxxx
Toll-free number: 0120-xxx-xxx
Expression including country code: +81-x-xxxx-xxxx

## Notes for Office 365
### 1. Full-width alphanumeric characters are converted to half-width alphanumeric characters beforehand.
When matching text in Office 365, full-width alphanumeric characters are converted to half-width alphanumeric characters before matching with regular expressions.
### 2. Handling half-width hyphens between full-width characters
Please note that half-width hyphens between full-width characters, such as "full-width - full-width", are replaced with "-" after spaces are inserted before and after the hyphen is matched.

## Phone Number Regular Expressions
### Employed Phone Number Regular Expressions
(?<!\w|[\---])0(\d([\---]| - )\d{3}|\d{2}([\---]| - )\d{2}|\d{3}([\---]| - )\d|\d{4}([\---]| - )|[5789]0([\---]| - )\d{3})\d([\---]| - )\d{4}(?!\w|[\---])

### Explanation of the above
#### \d
Indicates a single digit in half-width characters.
#### \d{4}
Indicates a four-digit number in half-width characters.
#### ([\－ー―-]| - )
Represents either a full-width or half-width hyphen, a full-width character similar to a hyphen, or a "-" after conversion as noted in Note 3. (Note: In the Office 365 confidential information definition, full-width hyphens now require escaping, so "-" has been changed to be represented as \-.)
#### [5789]
Represents one of the half-width characters 5, 7, 8, or 9.
#### \w
Equivalent to [A-Za-z0-9], representing a single half-width alphanumeric character.
#### (?<!\w|[\－ー―-])
This is a regular expression assertion that verifies that there are no half-width alphanumeric characters or hyphens before the rest of the string after matching. It excludes similar strings, such as serial codes like E03-0000-0000, that partially contain the format of a phone number. Since the assertion requires a fixed-length pattern, the previous definition has been revised.
#### (?!\w|[\---])
This is a regular expression assertion that checks for the absence of alphanumeric characters or hyphens after other parts match. It eliminates similar strings, such as serial codes like 03-0000-0000-0000, that partially contain phone number formats. Since the assertion requires a fixed-length pattern, we've revised the previous definition.

# Hyphen Type
| Character | UTF-8 | Unicode Code Point | Description | Applicable to This Feature |
|:---:|---:|---:|:---|:---:
| - | 2D | U+002D | Hyphen-Minus | x |
| - | E383BC | U+30FC | Katakana-Hiragana Prolonged Sound Mark | x |
| - | E28090 | U+2010 | Another Hyphen ||
| - | E28091 | U+2011 | Non-Breaking Hyphen ||
| - | E28092 | U+2012 | Figure Dash ||
| – | E28093 | U+2013 | En Dash ||
| — | E28094 | U+2014 | Em Dash ||
| ― | E28095 | U+2015 | Horizontal Bar | x |
| − | E28892 | U+2212 | Minus Sign ||
| ⁃ | E28183 | U+2043 | Hyphen Bullet ||
| ﹣ | EFB9A3 | U+FE63 | Small Hyphen-Minus ||
| ｰ | EFBDB0 | U+FF70 | Halfwidth Katakana-Hiragana Prolonged Sound Mark ||
| － | EFBC8D | U+FF0D | Fullwidth Hyphen-Minus | x |

# SIT3. Email Links
Detects email addresses in Office files that are used as mailto links, rather than text strings that are email addresses.
## Regular Expression
mailto:[\w\-.!#$%&'*+\/=?^_`{|}~]+@[\w\-_]+\.[\w\-_.]+

# SIT4. Japanese Calendar
## Detects the following Japanese calendar patterns.
December 12, 2019
May 1, 1988
Auspicious day in the 10th month of the year 2000
## Regular Expressions
(Meiji|Taisho|Showa|Heisei|Reiwa|Year)\s*([\d{1,4}|[元一二三四５６７８９十一二三十〇００]{1,4})\s*Year\s*([\d{1,2}|[元一二三四５６７８９十一二三十〇０]{1,2})\s*Month\s*([\d{1,2}|[元一二三四５６７８９十一二三十〇０〇]{1,2})\s*Day

# Reference Information
1. Check regular expressions online with the [Regular Expression Checker](http://okumocchi.jp/php/re.php)
1. You can check the character code using the [Unidode Character Tool](https://www.marbacka.net/msearch/tool.php#chr2enc)