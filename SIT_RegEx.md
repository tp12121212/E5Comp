# If a regular expression error occurs in a custom sensitive information definition
The Office 365 Data Classification Service currently uses the C++ Boost.Regex 5.1.3 engine.
[Reference: Before You Use a Custom Sensitive Information Type - Microsoft 365 Compliance | Microsoft Docs](https://docs.microsoft.com/ja-jp/microsoft-365/compliance/create-a-custom-sensitive-information-type?view=o365-worldwide#before-you-begin)
Regarding the regular expressions in custom sensitive information definitions, all manually registered custom sensitive information definitions are internally grouped into a common rule package. If an error occurs in any of the other regular expressions due to specification changes, all edits to custom sensitive information definitions that do not contain errors will also result in errors.

### Error displayed when a regular expression error prevents you from creating new sensitive information or updating existing sensitive information definitions.
This error message reads, "Client Error: The specified classification rule collection contains an invalid regular expression processor. The invalid regular expression processor identifier is "".
![SIT Error](https://github.com/YoshihiroIchinose/E5Comp/blob/main/Error_SIT.png)

If you receive an error like the one above, and the erroneous regular expression is contained in a single location, you can simply edit the sensitive information definition and return it to an error-free state. However, if the same regular expression is used in multiple sensitive information definitions, you cannot address these errors from the Compliance Center Web UI, as only one update can be made at a time. In such cases, you can extract the settings as XML using the following command, edit them all at once, and then re-upload them.

## Exporting confidential information definitions in bulk
```
#Connection ID/Password
$Id="admin@xxxx.onmicrosoft.com"
$Password = "xxxxx"
$outfile ="C:\Comp\SIT.xml"

#Generating Credentials
$SecPass=ConvertTo-SecureString -String $Password -AsPlainText -Force
$Credential= New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $Id, $SecPass

Connect-IPPSSession -UserPrincipalName $id -Credential $Credential
# Confidential information defined in the UI is collected into a single RulePack named "Microsoft.SCCManaged.CustomRulePack."
$r=Get-DlpSensitiveInformationTypeRulePackage |?{$_.LocalizedName -eq "Microsoft.SCCManaged.CustomRulePack"}
$r.ClassificationRuleCollectionXml |out-file $outfile
```
## Editing the XML
The extracted custom sensitive information definitions are saved in "C:\Comp\SIT.xml." Edit this XML file, delete or modify any erroneous sensitive information definitions as appropriate, and then save the file overwriting the previous one.

## Upload the edited XML again.
```
Set-DlpSensitiveInformationTypeRulePackage -FileData (Get-Content -Path $outfile -Encoding Byte)
```

## Reference: Regular Expression Error Confirmed on November 4, 2021
The following assertion, which was previously used in the regular expression for the phone number definition, now causes a new error.
### The first assertion, which results in an error, verifies that the found phone number is not preceded by alphanumeric characters, hyphens, or a combination of hyphens and spaces.
```
(?<!\w|[\－ー―-]|-)
```
### The last assertion, which results in an error, verifies that the found phone number is not followed by alphanumeric characters, hyphens, or a combination of hyphens and spaces.
```
(?!\w|[\－ー―-]| -)
```
When I investigated why the above was failing, I found that the [Perl Reference](https://www.boost.org/doc/libs/1_68_0/libs/regex/doc/html/boost_regex/syntax/perl_syntax.html) for Boost.Regex 5.1.3 states that Lookahead and Lookbehind must be fixed-length, and indeed, the above assertion does not return 1 It was discovered that the error occurred because one or two character patterns were used in both cases: letters and half-width spaces and half-width hyphens. In Office 365's DLP, when phone numbers consist of full-width numbers and half-width hyphens, the full-width numbers are not only converted to half-width numbers, but a pre-processing step is also performed to insert spaces between the full-width and half-width characters before matching. Therefore, if the original data is written using full-width numbers and half-width hyphens, there is a possibility that strings of an incorrect length for a phone number will be overdetected as a phone number. However, excluding this part, the correction required modifying the pattern to the one below, which excludes the half-width space + half-width hyphen case.
### Modified first assertion to ensure that the found phone number does not contain alphanumeric characters or hyphens before it.
```
(?<!\w|[\－ー―-])
```
### Modified last assertion to ensure that the found phone number does not contain alphanumeric characters or hyphens after it.
```
(?!\w|[\－ー―-])
```

### Modified phone number definition
The following is an example of a regular expression for a Japanese phone number. See also [Personal Information Definition for Japan](https://github.com/YoshihiroIchinose/E5Comp/blob/main/SIT.md).
```
(?<!\w|[\---])0(\d([\---]| - )\d{3}|\d{2}([\---]| - )\d{2}|\d{3}([\---]| - )\d|\d{4}([\---]| - )|[5789]0([\---]| - )\d{3})\d([\---]| - )\d{4}(?!\w|[\---])
```