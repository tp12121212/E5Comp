# Creating a Keyword Dictionary for Japanese Address Detection in EDM
This page explains how to create a keyword dictionary for Japanese address detection in EDM.

## Why a Keyword Dictionary is Needed for Japanese Address Detection in EDM
Detecting key elements in EDM requires defining a SIT (System Information Set) first and using that SIT to extract strings that serve as EDM match candidates.
Detecting Japanese addresses themselves can be easily accomplished by using regular expressions in the SIT or named entities.
However, when using EDM, the strings detected in the SIT are compared with hashed and imported data, so the ranges of the strings detected by the SIT and the address hash imported by EDM must strictly match. With regular expressions and named entities, it is difficult to control the extent of the address range extracted as a match candidate.
Fluctuation in the range of addresses detected by the SIT can result in a mismatch with the address hash imported by EDM.

With a keyword dictionary, the strings registered as keywords are generally detected as is, allowing you to precisely define the range of detection as an SIT.
When detecting addresses in EDM, you can use a keyword dictionary as an SIT like this,
and also prepare EDM address data within the range defined in this keyword dictionary to perform range-matching.

## How far should you register in a keyword dictionary?
Generally, when it comes to town numbers, house numbers, and block numbers, there is a wide range of variation, such as differences between kanji numerals and alphanumeric characters, whether to write them as a hyphen or chome, and whether or not to include apartment names.
For this reason, it is recommended to limit comparisons to prefecture + city/ward/town/village + town name, excluding chome numbers and apartment names.
When creating such a keyword dictionary, it is best to generate it by trimming the customer address data actually used in EDM.
However, for more general address detection, you can also register addresses identified by postal codes, as introduced on this page, in a keyword dictionary.
However, when importing data into EDM, you must first trim the address data to match the keyword dictionary and then hash it.

## Source Japan Post Address Information
Japan Post provides a list of addresses identified by postal codes in CSV format. Using this, we use a PowerShell script to generate a reproducible keyword dictionary for Japanese addresses.
The source data is downloaded in UTF-8 format, with one line per record.
[Postal Code Data Download](https://www.post.japanpost.jp/zipcode/download.html)

## Processing for the Keyword Dictionary
To create the keyword dictionary for SIT, which precedes EDM, we process Japan Post's postal code address data based on the following considerations.
1. The prefecture, city, town, or village name, which corresponds to columns 7-9 of the CSV data, are concatenated to form the address to be registered in the keyword dictionary.
1. Full-width alphanumeric characters are converted to half-width alphanumeric characters.
1. Even within the same address range, there are cases where () indicates different postal codes between the west and east sides, etc., but the part after () is ignored.
1. If the process in 3 results in duplicate addresses, do not register them in the keyword dictionary.
1. If town or village names sharing the same postal code are enclosed in ",", the prefecture and city, town, or village are stored and registered as separate lines in the keyword dictionary.
1. In cases where numbers are followed by "go," "sen," "ban," "cho," "ku," "jo," "dori," "irikai," "ranch," etc., or where numbers are followed by "dai," the number is converted to kanji numerals and the corresponding pattern is also registered in the keyword dictionary.
1. In Iwate Prefecture's land divisions, there are cases where multiple land divisions are combined into a single postal code using the characters ~. However, the numbers and land division parts are removed and registered in the keyword dictionary without duplicates.
1. Ignore any "if not listed below" in the town/district name and include only the prefecture, city, town, or village name.
1. Due to the above processing and the original address data, the existence of rows where partial addresses have a data inclusion relationship is allowed.

## Generated Keyword Dictionary
The keyword dictionary data generated using Japan Post data from February 29, 2024 is available here: [https://github.com/YoshihiroIchinose/E5Comp/blob/main/WB/JPAddressDic.txt]. However, this data alone does not support word-breaking that depends on the following string, so please also refer to the page here: [https://github.com/YoshihiroIchinose/E5Comp/blob/main/AddressDictionayforEDM2.md].

## Script body
```
#Create an array in advance so that $a[55] = "55"
$a=("1", "2", "3", "4", "5", "6", "7", "8", "9")
$b=("10", "11", "12", "13", "14", "15", "16", "17", "18", "19")
$a+=$b
for($i=2;$i -le 9; $i++){
Foreach($j in $b)
{
$a+=$a[$i]+$j
}
}
#Characters before and after the number
$suffix="([Line No. Block Section Article]|Street|Membership|Farm)"
$prefix="(Number)"

#If there is a land division, exclude the number and the number part.
Function IgnoreZiwari($original){
If($original -match '(\d{1,2})'+"地割"){
$temp=$original.Substring(0,$original.IndexOf($Matches[0]))
If($temp[$temp.length-1] -eq "第"){
Return $temp.Substring(0,$temp.length-1)
}
Return $temp
}
Return $original
}

# For partial addresses containing numbers, prepare a pattern replacing them with kanji numerals.
# Consider patterns where a single-digit number appears twice along with the ward or section.
Function variations($original){
$normalized=""
If($original -match '(\d{2})'+$suffix){
$normalized=$original.replace($Matches[0], $a[$Matches[1]]+$Matches[2])
}else{
if($original -match '(\d)'+$suffix){
$normalized=$original.replace($Matches[0], $a[$Matches[1]]+$Matches[2])
if($normalized -match '(\d)'+$suffix){
$normalized=$normalized.replace($Matches[0], $a[$Matches[1]]+$Matches[2])
}
}
}
If($original -match $prefix+'(\d)$'){
$normalized=$original.replace($Matches[0],$Matches[1]+$a[$Matches[2]])
}
If($normalized -eq ""){
Return $original
}
Else{
Return $original+"`r`n"+$normalized
}
}

#Splits when town district names are written together with " and "
Function SplitParallel($original){
$parts = $original.split(","")
If($parts.count -eq 1){return $original}
If($parts[1].length -eq 2){
$parts[1] = $parts[0].Substring(0,$parts[0].length-2) + $parts[1]
}
If($parts[1].length -eq 1){
$parts[1] = $parts[0].Substring(0,$parts[0].length-1) + $parts[1]
}
If($parts.count -eq 3){
$parts[2] = $parts[0].Substring(0,$parts[0].length-2) + $parts[2]
}
Return $parts
}

#Downloaded from Japan Post: 1 record, 1 line, UTF-8 Format data
$sourceFile="C:\WB\utf_ken_all.csv"
$source = New-Object System.IO.StreamReader($sourceFile, [System.Text.Encoding]::GetEncoding("utf-8"))

#Output file
$targetFile = "C:\WB\JPAddressDic.txt"
$target = New-Object System.IO.StreamWriter($targetFile, $false, [System.Text.Encoding]::GetEncoding("UTF-16LE"))

$prev=""
while (($line = $source.ReadLine()) -ne $null){
$data=$line.Split(",")
$address=$data[6]+$data[7]+$data[8]
$address=$address.Normalize([System.Text.NormalizationForm]::FormKC)
$address=$address.Replace("If not listed below","")
$address=$address.Replace("`"","")

If($data[8].IndexOf("follows") -ne -1){
If($data[8].Contains("if an address number is included")){continue}
If($data[8].Contains("if anything after an address number is included")){continue}
$address=$data[6]+$data[7]
}
$s=$address.LastIndexOf("(")
$e=$address.LastIndexOf(")")
If($e -> $s - and $s -> 0){
$address=$address.Substring(0,$s)
}
$address=IgnoreZiwari($address)
If($address.Equals($prev)){
continue
}
$prev=$address
Foreach($p in SplitParallel $address)
{
$nomalized=variations $p 
$target.WriteLine($nomalized) 
}
}
$source.Close()
$target.Close()
````