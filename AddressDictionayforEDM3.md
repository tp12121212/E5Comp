# Trim addresses imported into EDM to match the keyword dictionary
When matching the main elements of EDM, the corresponding strings are hashed and compared, so partial matches are not performed.
The string detected in the preceding SIT and the string hashed and imported by EDM must strictly match in range.

When detecting addresses as the main elements of EDM, one implementation method is to match addresses up to the prefecture, city, ward, town, village, or district name identified by the postal code,
to eliminate the effects of variations in the spelling of block numbers and the presence or absence of an apartment building name.
(However, if there are multiple EDM rows with the same prefecture, city, ward, town, or village name value, it will be necessary to further subdivide the addresses to reduce duplication.)

This script preprocesses the data to be imported into EDM. It uses the address keyword dictionary generated on [this page](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AddressDictionayforEDM2.md) to trim the address data to the prefecture, city, ward, town, or village name included in the keyword dictionary. However, if the EDM address data does not include the town or village name in the keyword dictionary due to aliases, spelling variations, or old address systems, the EDM data will be replaced with null characters. Therefore, if there are any blank address rows in the processed EDM data, you should consider updating the address data in the original corresponding row in the keyword dictionary.
This script also standardizes data by replacing full-width alphanumeric characters with half-width alphanumeric characters, as explained on [this page](https://github.com/YoshihiroIchinose/E5Comp/blob/main/EDM_Preprocess.md).

## Trimming Example: Replacing Address Data with Addresses up to the Street Name Registered in the Keyword Dictionary
#### Original Address Data from EDM
1-5-4 Katakura, Kanagawa Ward, Yokohama City, Kanagawa Prefecture
1-2 Kanagawa Honmachi, Kanagawa Ward, Yokohama City, Kanagawa Prefecture

#### Keyword Dictionary Contents
~
Oguchi-dori, Kanagawa Ward, Yokohama City, Kanagawa Prefecture
Oguchi-nakacho, Kanagawa Ward, Yokohama City, Kanagawa Prefecture
Oonomachi, Kanagawa Ward, Yokohama City, Kanagawa Prefecture
Katakura, Kanagawa Ward, Yokohama City, Kanagawa Prefecture
Kanagawa, Kanagawa Ward, Yokohama City, Kanagawa Prefecture
Kanagawa Honmachi, Kanagawa Ward, Yokohama City, Kanagawa Prefecture
~

### Converted Address Data
Katakura, Kanagawa Ward, Yokohama City, Kanagawa Prefecture
Kanagawa Honmachi, Kanagawa Ward, Yokohama City, Kanagawa Prefecture

## PowerShell Script
```
# CSV file of customer data to be imported into EDM (address information is in the Address (Column)
$sourceFile="C:\Data\EDMCustomers\CustomeTestData.csv"
# Destination for the converted CSV file
$targetFile = "C:\Data\EDMCustomers\CustomeTestData_Normalized.csv"
# Address keyword dictionary
$dictionaryFile = "C:\Data\JPAddressDicwithVariations.txt"

Function GetMaximumMatched($text){
$prev=""
Foreach($d in $dic){
Switch($text.CompareTo($d)){
1 {If($text.StartsWith($d)){$prev=$d}}
0 {return $text}
-1{return $prev}
}
}
return $prev
}

$dic=Get-Content $dictionaryFile
# It is important to sort the imported dictionary information to find the longest match.
$dic = $dic | Sort-Object | Where-Object {$_.StartsWith("_") -eq $false}
$csv=Import-Csv -Path $sourceFile

Foreach($c in $csv){
# Normalize full-width alphanumeric characters in the imported EDM data to half-width alphanumeric characters.
Foreach($att in $c|get-member|Where-Object{$_.MemberType -eq "NoteProperty"}){
$c.($att.Name)=$c.($att.Name).Normalize([System.Text.NormalizationForm]::FormKC)
}
# Trim the Address column value.
$c.Address=GetMaximumMatched $c.Address
}
$csv | Export-Csv -Path $targetFile -NoTypeInformation -Encoding unicode
```