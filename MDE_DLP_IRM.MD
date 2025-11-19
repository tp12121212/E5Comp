# Differences in file operation logs between each service
All of these services use the same sensor as Defender for Endpoint, which is represented by MSSense.exe. However, due to differences in monitoring purposes, the logs available in each service's UI differ.
- Defender for Endpoint: A system primarily designed to detect cyberattacks, specializing in monitoring processes, communications, the registry, and more.
- Endpoint DLP: A system that sets broad policies to monitor all users and monitors and controls general information export from devices.
- Insider Risk Management: A system that quantitatively analyzes the operations of users deemed high risk based on factors such as the absolute volume of file operations, planned retirement, and individual specifications. It primarily targets files containing confidential information, regardless of policies.

## Log output for a series of information leakage operations
| Operations | Defender for Endpoint Timeline | Activity Explorer in Endpoint DLP | Activity Explorer in Insider Risk Management |
| ---- | ------------ | ------------- | ------------- |
| 1. Downloading a file from an internal file server | 〇 Two logs recorded: explorer.exe created file xxx.txt<br>xxx.txt, file observed on host <br> (Condition: MDE-monitored file) | 〇 File created log recorded <br> (Condition: Endpoint DLP-monitored file + file with a policy-specified extension) | 〇 Sensitive file created log recorded <br> (Condition: Contains any sensitive information) | | | 2. Reading the downloaded file with compression software such as WinRAR.exe | × No specific log recorded | 〇 File accessed by unallowed app log recorded <br> (Condition: Accessing a file containing sensitive information specified in the DLP policy, but using an exe specified in Unallowed Apps) (Access from) | 〇 Records Sensitive File Read logs <br> (Condition: All exe files are targeted if they contain any sensitive information) |
| 3. Generating split RAR files from compression software such as WinRar.exe | 〇 Records WinRAR.exe created file xxx.part01.rar and WinRAR.exe created file xxx.part02.rar logs | 〇 Records File created xxx.part01.rar and File created xxx.part02.rar logs | × No specific logs |
| 4. Uploading RAR files to external websites | × No specific logs | 〇 Records FileUploadedToCloud logs <br> (Condition: Files monitored by Endpoint DLP + files with extensions specified by policy, uploaded via Edge or Chrome with the add-on) | 〇 Records File copied to cloud logs <br> (Condition: Via Edge or Chrome with the add-on) |
## Defender for Endpoint FileCreated File extensions observed in the event
+ .4io
+ .7z
+ .asax
+ .aspx
+ .bat
+ .bmp
+ .cfg
+ .cmd
+ .com
+ .config
+ .cookie
+ .crx
+ .css
+ .csv
+ .dat
+ .dat_
+ .dll
+ .DLL
+ .dll_
+ .dll_DiP_STAGED
+ .dmp
+ .docx
+ .dotx
+ .exe
+ .EXE
+ .exe_
+ .exe_DiP_STAGED
+ .gif
+ .htm
+ .html
+ .HTML
+ .inf
+ .js
+ .lnk
+ .LNK
+ .msi
+ .msp
+ .mui
+ .node
+ .pdf
+ .pem
+ .png
+ .pptx
+ .ps1
+ .ps1xml
+ .psd1
+ .psm1
+ .rar
+ .SFX
+ .sst
+ .svg
+ .tmp
+ .TMP
+ .txt
+ .wer
+ .xar
+ .xlsb
+ .xlsx
+ .zip

## Endpoint DLP Monitored Files
+ Word Files
+ PowerPoint Files
+ Excel Files
+ PDF Files
+ .csv Files
+ .tsv Files
+ .txt Files
+ .rtf Files
+ .c Files
+ .class Files
+ .cpp Files
+ .cs Files
+ .h Files
+ .java Files
+ .zip Files
+ .rar Files
+ .tar Files
+ .lzh Files

## Endpoint DLP Activity File extensions where FileUploadedToCloud was confirmed by specifying the extension in Explorer (probably all file types)
+ .cab
+ .csv
+ .docx
+ .dll
+ .exe
+ .jpg
+ .js
+ .lzh
+ .mp4
+ .pdf
+ .pfile
+ .pjpg
+ .png
+ .ptxt
+ .rar
+ .svg
+ .tar
+ .txt
+ .zip

## File extensions where Sensitive File Read and Sensitive File Created were confirmed in Insider Risk Management's Activity Explorer
+ .asd
+ .docx
+ .dotx
+ .xlsx
+ .csv
+ .pdf
+ .pptx
+ .txt
+ .xlsx

## File extensions where File Upload To Cloud was confirmed in Insider Risk Management's Activity Explorer (Probably all file types)
+ .cab
+ .csv
+ .docx
+ .dll
+ .exe
+ .jpg
+ .js
+ .lzh
+ .mp4
+ .pdf
+ .pfile
+ .pjpg
+ .png
+ .ptxt
+ .rar
+ .svg
+ .tar
+ .txt
+ .xlsx
+ .zip