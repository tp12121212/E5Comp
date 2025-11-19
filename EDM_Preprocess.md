# Converting full-width alphanumeric characters and symbols to half-width for EDM data upload
This PowerShell sample preprocesses a CSV file for EDM data upload, converting full-width alphanumeric characters and symbols to half-width, and converting half-width kana to full-width kana. The source file must be UTF-8 with BOM, UTF-16 LE, or UTF-16 BE; UTF-8 without BOM is not supported. The default file output is UTF-16 LE.

## Using the Normalize method
The following sample uses the Normalize method to perform bulk normalization on a row-by-row basis.
```
$source="E:\WorkData\Comp\EDM_Test\DB\EDM_CustomerDB.csv"
$target="E:\WorkData\Comp\EDM_Test\DB\EDM_CustomerDB_c.csv"

$output=@()
$lines=Get-Content $source
foreach($line in $lines){
$output+=$line.Normalize([System.Text.NormalizationForm]::FormKC)
}
$output|out-file $target
```
## Execution result
### Before conversion
```
ABCabc
ABCabc
03-3333-3333
03-3333-3333
Shinagawa Station
Shinagawa Station
#$%&
＃$%&
Aiueo
Aiueo
Aiueo
Papipupepo
Papipupepo
Dajizudedo
Dajizudedo
Yayuyo
Yayuyo
```
### After conversion
```
ABCabc
ABCabc
03-3333-3333
03-3333-3333
Shinagawa Station
Shinagawa Station
#$%&
#$%&
Aiueo
Aiueo
Aiueo
Papipupepo
Papipupepo
Dajizudedo
Dajizudedo
Yayuyo
Yayuyo
```

## Simple character replacement method
The following sample code shows how to simply replace each character. However, this does not take half-width kana into consideration. When replacing half-width kana, voiced and semi-voiced consonants are separate characters, and the replacement is two characters -> one character, so some ingenuity in implementation is required.
````
$source="E:\WorkData\Comp\EDM_Test\DB\EDM_CustomerDB.csv"
$target="E:\WorkData\Comp\EDM_Test\DB\EDM_CustomerDB_c.csv"

function ConvertTo-SingleBytes($str){
$chars=$str.ToCharArray()
$out = [char[]]::new($str.Length)
$count=0
foreach($char in $chars){ 
$out[$count]=switch -CaseSensitive($char){ 
'1' {'1';break} 
'2' {'2';break} 
'3' {'3';break} 
'4' {'4';break} 
'5' {'5';break} 
'6' {'6';break} 
'7' {'7';break} 
'8' {'8';break} '9' {'9';break} 
'0' {'0';break} 
'　' {' ';break} 
'A' {'A';break} 
'B' {'B';break} 
'C' {'C';break} 
'D' {'D';break} 
'E' {'E';break} 
'F' {'F';break} 
'G' {'G';break} 
'H' {'H';break} 
'I' {'I';break} 
'J' {'J';break} 
'K' {'K';break} 
'L' {'L';break} 
'M' {'M';break} 
'N' {'N';break} 
'O' {'O';break} 
'P' {'P';break} 
'Q' {'Q';break} 
'R' {'R';break} 
'S' {'S';break} 
'T' {'T';break} 
'U' {'U';break} 
'V' {'V';break} 
'W' {'W';break} 
'X' {'X';break} 
'Y' {'Y';break} 
'Z' {'Z';break} 
'a' {'a';break} 
'b' {'b';break} 
'c' {'c';break} 
'd' {'d';break} 
'e' {'e';break} 
'f' {'f';break} 
'g' {'g';break} 
'h' {'h';break} 
'i' {'i';break} 
'j' {'j';break} 
'k' {'k';break} 
'l' {'l';break} 
'm' {'m';break} 
'n' {'n';break} 
'o' {'o';break} 
'p' {'p';break} 
'q' {'q';break} 
'r' {'r';break} 
's' {'s';break} 
't' {'t';break} 
'u' {'u';break} 
'v' {'v';break} 
'w' {'w';break} 
'x' {'x';break} 
'w' {'y';break} 
'z' {'z';break} 
'! ' {'!';break} 
'#' {'#';break} 
'$' {'$';break} 
'%' {'%';break} 
'&' {'&';break} 
'^' {'^';break} 
'\' {'\';break} 
'@' {'@';break} 
';' {';';break} 
':' {':';break} 
',' {',';break} 
'． ' {'.';break} 
'/' {'/';break} 
'=' {'=';break} 
'~' {'~';break} 
'|' {'|';break} 
'''' {'';break} 
'{' {'{';break} 
'+' {'+';break} 
'＊' {'*';break} 
'}' {'}';break} 
'<' {'<';break} 
'＞' {'>';break} 
'? ' {'?';break}
'＿' {'_';break}
'﹣' {'-';break}
'－' {'-';break}
default {$char}
}
$count++
}
return [String]::new($out)
}

$output=@()
$lines=Get-Content $source
foreach($line in $lines){
$output+=ConvertTo-SingleBytes $line
}
$output|out-file $target
```

## Execution result
### Before conversion
```
ABCabc
ABCabc
03-3333-3333
03-3333-3333
Shinagawa Station
Shinagawa Station
#$%&
＃＄％＆
Aiueo
Aiueo
Aiueo
Papipupepo
Papipupepo
Dajizudedo
Dajitsudedo
Yayuyo
Yayuyo
```
### After conversion
```
ABCabc
ABCabc
03-3333-3333
03-3333-3333
Shinagawa Station
Shinagawa Station
#$%&
#$%&
Aiueo
Aiueo
Aiueo
Papipupepo
Papipupepo
Dajizudedo
Dajitsudedo
Yayuyo
Yayuyo
```