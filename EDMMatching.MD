# Regarding confidential information and Exact Data Match
This page summarizes the results of various operational tests in the Microsoft 365 environment, particularly regarding matching of custom confidential information definitions in a Japanese environment, and the mechanism of Exact Data Match.

## Regarding custom confidential information matching
When matching regular expressions or custom confidential information, the following preprocessing is roughly performed on the original data at the time of text extraction.
- Full-width alphanumeric characters, full-width symbols, and full-width spaces are converted to the corresponding half-width alphanumeric characters and half-width spaces.
- Half-width kana characters are converted to full-width kana characters.
- When extracting text in Word, if a full-width number is adjacent to a full-width character, not only is it converted to a half-width number, but a space is inserted between them.

| Original Data | Annotations | Text Extraction in UTF-8 .txt Format with BOM | Text Extraction in Word in .docx Format |
| ---- | ---- | ---- | ---- |
| Tokyo 2020 | Half-width Numbers as Is | Tokyo 2020 | Tokyo 2020 |
| Tokyo 2020 | Full-width Numbers Converted to Half-width Numbers | Tokyo 2020 | Tokyo 2020 |
| ABC Analysis | Half-width Alphabet Characters as Is | ABC Analysis | ABC Analysis |
| ABC Analysis | Full-width Alphabet Characters Converted to Half-width Alphabet Characters | ABC Analysis | ABC Analysis |
| Tokyo Toshinagawa Champion | Half-width kana converted to full-width kana | Tokyo Toshinagawa Champion | Tokyo Toshinagawa Champion |
| "　" | Full-width spaces converted to half-width spaces | " " | " " |
| """' | These full-width symbols remain unchanged | """'' | """'' |
| !＃＄％＆（）－＾￥＠；：，。／＝～｜‘｛＋＊＜＞？＿ | These full-width symbols converted to half-width | !#$%&()-^¥@;:,./=~\|‘{+*}<>?_ | !#$%&()-^¥@;:,./=~\|‘{+*}<>?_ |

## Hyphen conversion
| Character | Unicode Code Point | Description | UTF-8 .txt with BOM Text extraction in .docx format | Text extraction in Word | Treated as a hyphen in credit card SIT? |
|:---:|---:|---:|:---|:---:|:---:|
| - | 2D | Hyphen-Minus | - (Convert to 2D half-width hyphen) | - (Convert to 2D half-width hyphen) | Yes |
| ﹣ | FE63 | Small Hyphen-Minus | - (Convert to 2D half-width hyphen) | - (Convert to 2D half-width hyphen) | Yes |
| － | FF0D | Fullwidth Hyphen-Minus | - (Convert to 2D half-width hyphen) | - (Convert to 2D half-width hyphen) | Yes |
| - | 2010 | Another Hyphen | - (2010 unchanged) | - (2010 unchanged) | No |
| ‑ | 2011 | Non-Breaking Hyphen | ‐ (Convert to 2010) | ‐ (Convert to 2010) | No |
| ‒ | 2012 | Figure Dash | ‐ (2012 unchanged) | ‐ (2012 unchanged) | No |
| – | 2013 | En Dash | ‐ (2013 unchanged) | ‐ (2013 unchanged) | No |
| — | 2014 | Em Dash | ‐ (2014 unchanged) | ‐ (2014 unchanged) | No |
| ― | 2015 | Horizontal Bar | ― (2015 unchanged) | ― (2015 unchanged) | No |
| ⁃ | 2043 | Hyphen Bullet | ⁃ (2043 unchanged) | ⁃ (2043 unchanged) | No |
| − | 2212 | Minus Sign | − (2212 unchanged) | − (2212 unchanged) | No |
| − | 30FC | Katakana-Hiragana Prolonged Sound Mark | − (30FC unchanged) | − (30FC unchanged) | No |
| − | FF70 | Halfwidth Katakana-Hiragana Prolonged Sound Mark | − (Convert to 30FC) | − (Convert to 30FC) | No |

Based on the above, the above single hyphen character can be found in Microsoft 365 using the following regular expression:
```
[-|\x{2010}-\x{2015}|\x{2043}|\x{2212}|\x{30FC}]
```

This can be confirmed using the Test-TextExtraction command on Exchange Online, as shown below.
Note that Exchange Online uses UTF-8 without BOM. This tool does not support text in ASCII or ANSI (Shift-JIS) format. Specifying text in these formats will result in an error indicating that the tool cannot parse the document. (As of May 23, 2022) Also, because SharePoint Online and Endpoint DLP use different engines, behavior may differ slightly.
```
Connect-ExchangeOnline
$content = Test-TextExtraction -FileData ([System.IO.File]::ReadAllBytes("E:\WorkData\Comp\TestData.docx"))
$content.ExtractedResults[0].ExtractedStreamText
```
## Notes on Specifying Regular Expressions and Keywords
As mentioned above, full-width alphanumeric characters and symbols are converted to half-width alphanumeric characters and symbols, and half-width kana characters are also converted to full-width kana characters during text extraction. Therefore, when defining custom confidential information using regular expressions or keywords,
you do not need to be aware of the existence of full-width alphanumeric characters, full-width symbols, or half-width kana characters. Simply defining the characters using half-width alphanumeric characters, half-width symbols, and full-width kana characters will ensure complete matching.
However, since "Tokyo 2020" is often extracted as "Tokyo 2020" in Word files, it is important to consider the possibility of half-width spaces being inserted when full-width numbers are converted to half-width numbers.

## About EDM Data Preparation and Hash Generation
EDM During hash generation and data import, full-width alphanumeric characters are not treated the same as half-width alphanumeric characters, and half-width kana characters are not converted to full-width kana characters. Therefore, all full-width alphanumeric characters and full-width spaces in the imported data must be converted to half-width alphanumeric characters and half-width spaces in advance, and half-width kana characters must also be converted to full-width kana.
[Reference: Script to convert full-width alphanumeric characters and symbols to half-width characters for EDM data upload.](https://github.com/YoshihiroIchinose/E5Comp/blob/main/EDM_Preprocess.md)
Furthermore, while keywords consisting of multiple words with spaces can be used to detect primary elements, they cannot currently be detected as secondary elements. As described below, a custom confidential information definition is required as a prerequisite, just like primary elements. As shown in the test results below, you can set the schema to ignore half-width kana characters and treat half-width English characters as case-insensitive, but it does not convert full-width English characters to half-width characters or full-width spaces to half-width spaces.
```
Schema
caseInsensitive="true"
ignoredDelimiters=" ,',-,.,_,}​​​​​​​​,{​​​​​​​​,/,\,!,+,(,)"

How to verify
Generate a hash of the data to be imported using EdmUploadAgent.exe /CreateHash, and test whether the hash is different or identical depending on the notation.

〇 Half-width spaces can be ignored in the schema settings.
Hash("Yamada A Taro") = Hash("Yamada A Taro")

〇 Case insensitivity can be ignored in the schema settings.
Hash("Yamada A Taro") = Hash("Yamada A Taro")

× Half-width and full-width spaces are not considered equivalent.
(Full-width spaces are not ignored even if added to ignoredDelimiters.)
Hash("Yamada A Taro") ≠ Hash("Yamada A Taro")
Hash("Yamada Taro") ≠ Hash("Yamada Taro")

× Half-width and full-width alphanumeric characters are not considered equivalent.
Hash("Yamada A Taro") ≠ Hash("Yamada A Taro")
Hash("Yamada A Taro") ≠ Hash("Yamada A Taro")
Hash("Yamada 1 Taro") ≠ Hash("Yamada 1 Taro")

× Half-width and full-width kana characters are not considered equivalent.
Hash("Sato Takashi") ≠ Hash("Sato Takashi")
```

## About Exact Data Match
Exact Data Match uses the following three methods: Detection and confidence assessment are performed according to the definition of each step.
- A. Define the type of sensitive information to extract text that can be matched with the main element of Exact Data Match.

Custom definitions can be made using regular expressions, or predefined sensitive information such as credit card numbers and My Number can be used.
- B. Hash of the searchable main element as Exact Data Match.
- C. Hash of the auxiliary element as Exact Data Match.

### Based on the above definition, Exact Data Match works as follows:

1. From the target data, full-width alphanumeric characters and full-width symbols are converted to half-width alphanumeric characters and half-width symbols, and half-width kana characters are converted to full-width kana to extract text.

2. Use the type of sensitive information A to find the target pattern.

When detecting this pattern A, you can also filter the target using auxiliary elements such as keywords in the definition A.
Even in this case, only the main element of A is used to match the main element of Exact Data Match.

3. From the found pattern, Exact Data Match is performed. According to the schema definition, remove ignored characters such as spaces and hyphens to generate a hash.
4. Compare the hash of the main element captured in B with the hash of 3. If they are the same, the main element is successfully matched.
5. If matching of the auxiliary element C is also required, the strings before and after the main element are examined.
The strings before and after are word-broken and a hash is generated for each broken word.
6. Compare the hash of the auxiliary element C with the hash of 5. If the hashes match, the auxiliary element is successfully matched.
7. According to the confidential information definition of Exact Data Match, the detection result is returned with a confidence level based on the detection results of the main and auxiliary elements.

### The following points and considerations apply to this operation:
- In the text to be matched, full-width alphanumeric characters and full-width symbols are converted to half-width characters, as in normal confidential information detection,
and half-width kana characters are also converted to full-width kana. Therefore, even in Exact Data Match, half-width alphanumeric characters, half-width symbols, and full-width kana characters are converted to full-width characters.You only need to prepare the data with hyphens and spaces in mind.
- The detection of confidential information in the previous step, step A, does not take into account the ignored symbols defined in the EDM schema,
so you need to prepare regular expression patterns that take into account both hyphens and spaces and those that do not.
- Sensitive information detection works across Microsoft 365 once you define the type of sensitive information, regardless of whether a DLP policy is in place,
and confidential information is identified. Therefore, it is best to avoid defining A with content that targets broad patterns of text.
If the target is too general, processing may be skipped. It is also effective to narrow the target by adding auxiliary elements to the definition of A,
for example. - Half-width kana characters are converted to full-width kana characters for matching, but in half-width kana, voiced and semi-voiced consonants are counted as one character.
When converting to full-width kana, the position of the characters may not match.
There is no problem matching half-width kana characters with full-width kana keywords in normal confidential information definitions. However, when targeting half-width kana characters with Exact Data Match, half-width kana characters containing voiced and semi-voiced consonants currently do not find matches. (As of May 23, 2022)
- When matching auxiliary elements in Exact Data Match, word breaking is not performed when importing hashes, and one hash is imported per column. However, when scanning auxiliary elements, Japanese documents are word-broken and the hash comparison is performed.
As a result, auxiliary elements such as three-character kanji characters that can be word-broken will not be found with Exact Data Match. (As of May 23, 2022)

Examples of elements found as auxiliary elements: company employee, Sato, Takashi, Suzuki

Examples of elements not found as auxiliary elements: Ijuin, Takanashi, accountant

This is because the text is broken into multiple words, such as Iju/in, Takanashi/yu, and accountant/shi.

A hash is generated and compared for each broken word,

but the imported hashes are for "Ijuin," "Takanashi," and "accountant."

Since the word boundaries do not match, an identical hash cannot be found, resulting in a determination that the auxiliary element is not found.

When detecting searchable columns and main elements in Exact Data Match,

the target text is extracted using regular expressions according to the confidential information definition in the previous step, A,

and a hash is generated and compared. This problem does not occur because it does not rely on word breaking.

To address this issue, instead of hashing using word-broken words in step 5 above, you can define patterns for the auxiliary elements separately using keyword dictionaries, regular expressions, etc., and generate hashes using the patterns found. This can be done by comparing the results. However, this requires PowerShell operations. Download the EDM confidential information definition XML, specify the preceding custom confidential information definition in the Match tag of the auxiliary element, as well as the idMatch tag of the main element, and then re-upload it. For details, see page 23 of [here](https://microsoft.github.io/ComplianceCxE/resources/files/Deploying%20EDM%20for%20accurate%20classification%20v2%20(2).pdf). ````
Connect-IPPSSession
$def = Get-DlpSensitiveInformationTypeRulePackage -Identity "EDM Customer DB"
[System.IO.File]::WriteAllBytes('C:\EDM\EDM Customer DB.xml', $def.SerializedClassificationRuleCollection)
(Edit XML)
Set-DlpSensitiveInformationTypeRulePackage -FileData ([System.IO.File]::ReadAllBytes('C:\EDM\EDM Customer DB.xml'))
````