# Addressing word-break offsets in word matching with a keyword dictionary
Even if you register the keyword dictionary created on the previous page (https://github.com/YoshihiroIchinose/E5Comp/blob/main/AddressDictionayforEDM.md) in your keyword dictionary, there will still be addresses in the text that are not detected. This is because, while it is generally assumed that a block number follows the town area in an address, the word-break position can be offset depending on whether or not the block number is included.

## Word-break offsets due to the presence or absence of a block number
Example: String registered in a keyword dictionary
Ⓐ Kawada-cho, Ichinomiya City, Aichi Prefecture

Example address in the text
Ⓑ Kawada-cho 1, Ichinomiya City, Aichi Prefecture
Ⓒ 1-2-3 Kawada-cho, Ichinomiya City, Aichi Prefecture
Ⓓ Kawada-cho, Ichinomiya City, Aichi Prefecture

The following word-breaking is performed for each of these.
A' Kawada-cho, Ichinomiya City, Aichi Prefecture
B' Kawada-cho, Ichinomiya City, Aichi Prefecture 1
C' Kawada-cho, Ichinomiya City, Aichi Prefecture 1 2 3
D' Kawada-cho, Ichinomiya City, Aichi Prefecture, but

In the above case, A', B', C', and D' have different word breaks. Therefore, in a keyword dictionary that uses word matching,
even if you register the string A in the keyword dictionary, the word A cannot be detected from B, C, and D in the text.

Word-breaking results using a custom app
(Verifying word-breaking requires creating an app, and calling word-breaking is not easy.)
![image](https://github.com/YoshihiroIchinose/E5Comp/assets/66407692/d97f60e4-618c-4d80-a732-f0d2302e0fbc)

## Total word-breaking deviations for addresses identified by postal code
The following analysis results, using a custom app with Microsoft 365 word breakers, show the amount of word-breaking deviations for addresses identified by postal code created on the previous page. While 85.65% of addresses are consistent regardless of whether or not they have a block number, the remaining 15% or so have different word breaks within the patterns B through D without a block number. In the four-digit determination results below, the numbers from left to right correspond to word breaks A, B, C, and D, respectively. Pattern A is assigned a value of 0, and word breaks B through D are compared with each other. If a common word break pattern appears, the same number is assigned, and new word break patterns are assigned a new number +1. Therefore, if all word breaks A through D match, the result is "0000." If only B differs in the word break, the result is "0100." If A and B have the same pattern but C and D have the same pattern, the result is "0011." If A through D are all different, the result is "0123." The aforementioned Kawada-cho area in Ichinomiya City, Aichi Prefecture, is one of only 35 "0123" patterns out of the 120,000 addresses identified by postal codes nationwide.

Reference: [Analysis Results Excel](https://github.com/YoshihiroIchinose/E5Comp/blob/main/WB/%E4%BD%8F%E6%89%80%E8%BE%9E%E6%9B%B8_d_ana_M365.xlsx)

![image](https://github.com/YoshihiroIchinose/E5Comp/assets/66407692/1041f6d1-ec7d-43e6-a199-b5dbf15ff547)

## Keyword Dictionary Handling
In just under 15% of addresses, there are variations in word break locations depending on whether or not a block number is included. To detect these variations, you can register keywords that anticipate word breaks in your keyword dictionary.
One way to do this is to split keywords into the words you want to detect and register them linked with "\_" (before and after).

In the previous example, in addition to the standard flat address description of pattern A, you would also register the following string assuming B and C.

Aichi Prefecture, Ichinomiya City, Kawada-cho
\_Aichi_Prefecture_Ichinomiya_City_Kawada_cho_
\_Aichi_Prefecture_Ichinomiya_City_Kawada_cho_
\_Aichi_Prefecture_Ichinomiya_City_Kawada_cho_

Note that when the above string is word-breaked for word matching, the "_" is removed, resulting in the expected word-breaking.

Results of word breaking using a custom app
![image](https://github.com/YoshihiroIchinose/E5Comp/assets/66407692/243f7116-0036-46c4-b3fd-a7bb4b20578d)

## Address dictionary reflecting Microsoft 365 word breaking offsets
Based on the address dictionary generated on the previous page, [https://github.com/YoshihiroIchinose/E5Comp/blob/main/AddressDictionayforEDM.md], a Japanese address dictionary has been added with addresses with different word breaking positions by inserting "\_" characters. [Here](https://github.com/YoshihiroIchinose/E5Comp/blob/main/WB/JPAddressDicwithVariations.txt)
This address dictionary is UTF-16 encoded and 3.68MB in size. However, when imported using PowerShell, it fits within the 1MB keyword dictionary size limit after compression.

## About Updating a Keyword Dictionary via the UI
There is no problem defining a new keyword dictionary in the UI or PowerShell using the "\_" separator method described above, but you cannot update the keyword dictionary via the UI. This is because when loading existing registered keywords, the UI displays the string with the "\_" characters removed. If you update and save the dictionary as is, both the "\_" separator and the unseparated string will be saved as the same string, and the explicit word break specification will be lost. Therefore, when managing such dictionaries, you must update the entire dictionary via PowerShell.

## Importing an Address Dictionary via PowerShell
Using [Exchange Online PowerShell](https://learn.microsoft.com/ja-jp/powershell/exchange/connect-to-exchange-online-powershell?view=exchange-ps), you can import an address dictionary using the following PowerShell command:
```
Connect-IPPSSession
$fileData = [System.IO.File]::ReadAllBytes("C:\WB\JPAddressDicwithVariations.txt")

#To create a new dictionary
New-DlpKeywordDictionary -Name "JPAddress" -Description "Postal Code-Based Japanese Address" -FileData $fileData

#To update a dictionary
Set-DlpKeywordDictionary -Identity "JPAddress" -FileData $fileData
```