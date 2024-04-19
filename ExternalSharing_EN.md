# How to securely share files with the outside of the company

This is a summary of the secure file sharing methods available in Office 365 instead of PPAP. For 2 to 5, it is accompanied by file encryption using AIP. For conditions for external users to open AIP-protected files, [here](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AIP_FAQ.MD).

| Method | Overview | No need for external accounts | Secondary use restrictions on files | Supported files |
|:---|:---|:---|:---|:---|
| Anonymous link with password by 1 SharePoint Online / OneDrive for Business | Unique URL for file access for those who share and know the URL How to share files without user authentication. You can also set a password for file access. If it is an Office file, you can also set the download ban. | Not required | Not required | Anonymous links have no file restrictions. The download ban is limited to Office files. <br> In the case of downloading prohibited, Office for the Web will be used, so Excel files can be supported up to 100MB. <br>[Restriction on the file size of the book in SharePoint](https://support.microsoft.com/en-us/office/9e5bc6f8-018f-415a-b890-5452687b325e)|
| 2 Individually attach AIP protected files to e-mails and send them | Encrypt Office files with AIP with the recipient's e-mail address | Required | Possible | ・Word (doc, Docx, docm, dot, dotx dotm)<br>・Excel(xls, xlsx, xlsm, xlt, xltx, xltm, xlsb)<br>・PowerPoint (ppt, ppt X, pptm,potx, potm, pps, ppsx, ppsm)<br>・Visio (vsdm,.vsdx,vssm,vssx,vstm,vstx)<br>・XPS<br>・PDF<br>・ pFile (bpm, gif, jfif, jpe, jpeg, jpg, jt, png, tif, tiff, txt, xla, xlam, xml)<br>[Reference: Protection is supported Type of file](https://docs.microsoft.com/en-us/azure/information-protection/rms-client/clientv2-admin-guide-file-types#file-types-supported-for-protection)<br> For Outlook/Exchange Online, the maximum size of the attachment is 150MB In the case of Outloo on the Web, 112MB (taking 33% overhead from 150MB) [Message limit](https://docs.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-limits#message-limits)|
| 3 B2B sharing using the IRM library on SharePoint Online | IRM protection is automatically applied when you invite external users individually and download it on the site Share files through Burari. You can also set the validity period after downloading the file in the IRM library settings. | Required <br>・Office 365 Tenant ID<br>・If the B2B integration function is enabled, authentication with one-time password is also possible | Possible | ・Word, Excel, PowerPoint Open XML format and old format <br>・InfoPath<br>・XPS<br> [Reference: The operation of IRM for lists and libraries](https://docs.microsoft.com/en-us/microsoft-365/compliance/apply-irm-to-a-list-or-library?view=o365-worldwide#how-irm-works-for-lists-and-libraries)|
| 4Message Encryption | In the email option encryption setting in Outlook/Exchange Online, select Untransferable, and then add an Office file that supports encryption I'll attach it and send it. Not only Office files, but also administrator settings can protect PDFs. In the template setting of Advanced Message Encrpytion, even if the other party is an Office 365 user, it does not send encrypted files and emails directly OME (Office 365 Mes It is also possible to redirect to the sage Encryption portal. | Not required<br>・If the recipient is an Office 365 tenant, emails and attachments will be sent in an AIP encrypted state<br>・If the recipient is not an Office 365 tenant, OME (Office 365 Message Encryption) is redirected to the portal, and after one-time passcode authentication on the dedicated site, the content is displayed in the browser <br> and the corresponding files are downloaded When loaded, it will be encrypted with AIP <br> | Possible | ・Word(doc, docx, docm, dot, dotx dotm)<br>・Excel(xls, xlsx, xlsm , xlt, xltx, xltm, xlsb, xla, xlam)<br>・PowerPoint(ppt, pptx, pptm, pot, potx, potm, pps, ppsx, ppsm , thmx)<br>・IntoPath(xsn)<br>・XPS<br>・PDF(Activation setting required)<br>[Reference: How to use IRM in email messages](https://support.microsoft.com/en-us/office/bb643d33-4a3f-4ac7-9770-fd50d95f58dc)<br>Up to 25MB including the attached file of the email body. <br>[Is there a limit to the size of the message that can be sent with OME?](https://docs.microsoft.com/en-us/microsoft-365/compliance/ome-faq?view=o365-worldwide#ome--------------------------------)<br>|
| Session control with 5Defender for Cloud Apps | Web access to managed SaaS apps that can configure authentication such as Azure AD and SSO integration, De Apply the AIP secret label when downloading the file or prohibit the download via the reverse proxy of fender for Cloud Apps. You can use an existing SharePoint Online site, but you need a file sharing site that can be accessed by external users. | Necessary | Possible | ・Word (docm, docx, dotm, dotx)<br>・Excel (xlam, xlsm, xlsx, xltx)<br>・PowerPoint (potm, potx, ppsx , ppsm, pptm, pptx)<br>・PDF<br>[Reference: Protect the file when downloading](https://docs.microsoft.com/en-us/defender-cloud-apps/session-policy-aad#protect-download)<br> AIP can protect up to 50MB of files. [Apply the label directly to the file](https://docs.microsoft.com/en-us/defender-cloud-apps/azip-integration#how-to-integrate-azure-information-protection-with-defender-for-cloud-apps) |
<br>

# 1 Anonymous link with password by SharePoint Online / OneDrive for Business
### Tenant requirements
Anonymous links must be allowed in SharePoint Online/OneDrive for Business
### Overview of settings
After uploading the file to SharePoint Online /OneDrive for Business, choose Share -> "All users who know the link" from the file menu, and then pa Go to the Seward settings, uncheck "Allow editing", and then enable "Prohibit downloads".
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/FS_1.png">
[Reference: OneDrive file and folder sharing](https://support.microsoft.com/en-us/office/9fcc2f7d-de0c-4cec-93b0-a82024800c07)
<br>
<br>
# 2 Send individually AIP-protected files attached to an email
### Tenant requirements
Either license, including Office 365 E3 or Azure Informatino Protection P1, is required to protect files in AIP
### Overview of settings
Use Office's Information Rights Management function and Unified Label AIP Client to set permissions that specify the email address of external users. Generate an AIP-protected file in advance, attach the file to an email and send it.
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/IRM2.png">
[Reference: Restrict access to presentations using Information Rights Management PowerPoint](https://support.microsoft.com/en-us/office/bdbc7b2f-ea79-4e77-93fa-de49d48b3567)
<br>
<br>
# 3 B2B sharing using the IRM library in SharePoint Online
### Tenant requirements
・External invitations must be allowed in SharePoint Online
・Any license including Office 365 E3 or Azure Informatino Protection P1 is required to use the IRM library
### Overview of settings
In the library settings of the target SharePoint Online site, enable the IRM (Information Rights Managment) setting and upload the corresponding file I do. With this setting, a protected file with a permission setting that can only be opened by the person at the time of file download is generated each time. With the option setting, you can also set the upload of files that do not support, the validity period after downloading the file, and the permissions of the additional group to be granted.
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/IRM1.png">
[Reference: Apply Information Rights Management to the list or library](https://support.microsoft.com/en-us/topic/3bdb5-C4e-94fc-4741-b02f-4e7cc3c54aa1)
<br>
<br>
# 4Message Encryption
### Tenant requirements
・The use of standard Message Encrption requires either Office 365 E3 or Azure Informatino Protection P1.

・Advanced Message Encryption requires a license of either Microsoft 365 E5 / Office 365 E5 / E5 Complinace / IP&G
### Overview of settings
In Outlook connected to Exchange Online, when creating a new email, select Untransferable from the optional encryption setting, attach the file and send it. If the recipient is an Office 365 tenant, emails and attachments will be sent in an AIP encrypted state. If the recipient is not an Office 365 tenant, it will be redirected to the OME (Office 365 Message Encryption) portal, and after one-time passcode authentication on the dedicated site, You can display the content in the browser. If you download the corresponding file from the OME portal, it will be encrypted with AIP.
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/FS_2.png">
[Reference: Message encryption](https://docs.microsoft.com/en-us/microsoft-365/compliance/ome?view=o365-worldwide)
<br>
<br>
# 5Defender for Cloud Apps
### Tenant requirements
Defender for Cloud Apps requires a license that includes Defender for Cloud Apps

Requires a license including Azure Active Directory P1 for conditional access control

A license including Azure Informatino Protection P1 is required for file protection in AIP

### Overview of settings
Focusing on the following supported apps, activate session restrictions under certain conditions with Azure AD conditional access after SSO settings with Azure AD by the tenant administrator. In the settings of Defender for Cloud Apps, define the session policy and define what kind of control you want to do while using conditions such as the contents of the file and the extension of the file. With these settings, when downloading the corresponding file under the session limit, you can dynamically protect the file with the confidentiality label or the permission setting limited to the user who downloaded it. It is also possible to block the download itself while allowing reference and editing by Office for the Web.
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/MDCA1.png">
[Reference: Protect files when downloading](https://docs.microsoft.com/en-us/defender-cloud-apps/session-policy-aad#protect-download)

Session control compatible app

- AWS
- Azure DevOps (Visual Studio Team Services)
- Azure portal
- Box
- Concur
- CornerStone on Demand
- DocuSign
- Drop box
- Dynamics 365 CRM (Preview)
- Egnyte
- Exchange Online
- GitHub
- Google Workspace
- HighQ
- JIRA/Confluence
- OneDrive for Business
- LinkedIn Learning
- Power BI
- Salesforce
- ServiceNow
- SharePoint Online
- Slack
- Tableau
- Microsoft Teams (Preview)
- Workday
- Workiva
- Workplace by Facebook
- Yammer (Preview)

In addition, even applications other than the above can be supported if you can change the authentication method with SAML 2.0 or Open ID Connect.

[Reference: Supported apps and clients](https://docs.microsoft.com/en-us/defender-cloud-apps/proxy-intro-aad#featured-apps)