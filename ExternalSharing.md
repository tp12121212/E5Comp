# Secure File Sharing Methods with External Parties
This article summarizes secure file sharing methods available with Office 365 that replace PPAP. ​​Methods ② through ⑤ require file encryption using AIP. For information on the conditions for external users opening AIP-protected files, see [here](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AIP_FAQ.MD).

| Method | Overview | External Account Required? | Restrictions on Secondary File Use | Supported Files |
|:---|:---|:---|:---|:---|
| ① Anonymous Link with Password via SharePoint Online / OneDrive for Business | This method involves sharing a unique URL for file access with anyone who knows the URL, without requiring user authentication. You can also set a password for file access. For Office files, you can also prohibit downloads. | Not Required | Not Allowed | There are no file restrictions for anonymous links. Download prohibition is limited to Office files. If downloads are prohibited, Excel files will be limited to a maximum size of 100MB due to the use of Office for the Web. [SharePoint Workbook File Size Limits](https://support.microsoft.com/ja-jp/office/9e5bc6f8-018f-415a-b890-5452687b325e) |
| ② Send individually AIP-protected files as email attachments | Encrypt Office files using AIP by specifying the recipient's email address | Required | Possible | ・Word (doc, docx, docm, dot, dotx, dotm)<br>・Excel (xls, xlsx, xlsm, xlt, xltx, xltm, xlsb)<br>・PowerPoint (ppt, pptx, pptm, potx, potm, pps, ppsx, ppsm)<br>・Visio (vsdm,.vsdx,vssm,vssx,vstm,vstx)<br>・XPS<br>・PDF<br>・pFile (bpm, gif, jfif, jpe, jpeg, jpg, jt, png, tif, tiff, txt, xla, xlam, xml)<br>[Reference: File types supported for protection](https://docs.microsoft.com/ja-jp/azure/information-protection/rms-client/clientv2-admin-guide-file-types#file-types-supported-for-protection)<br>The maximum attachment size for Outlook/Exchange Online is 150MB, and for Outlook on the Web it is 112MB (taking into account a 33% overhead from 150MB). [Message Limits](https://docs.microsoft.com/ja-jp/office365/servicedescriptions/exchange-online-service-description/exchange-online-limits#message-limits) |
| ③ B2B Sharing Using an IRM Library in SharePoint Online | Invite external users individually and share files through an IRM library, where AIP protection is automatically applied when the files are downloaded from within the site. You can also set the expiration period for files after they are downloaded in the IRM library settings. | Required<br>・Office 365 tenant ID<br>・One-time password authentication is also possible if B2B integration is enabled | Possible | ・Word, Excel, and PowerPoint Open XML and legacy formats<br>・InfoPath<br>・XPS<br> [Reference: IRM for Lists and Libraries](https://docs.microsoft.com/ja-jp/microsoft-365/compliance/apply-irm-to-a-list-or-library?view=o365-worldwide#how-irm-works-for-lists-and-libraries)|
| ④ Message Encryption | In the encryption settings for Outlook/Exchange Online email options, select Do Not Forward and send an encrypted Office file as an attachment. In addition to Office files, administrator settings can also protect PDFs. By configuring the Advanced Message Encryption template, you can redirect recipients to the OME (Office 365 Message Encryption) portal instead of directly sending encrypted files or emails, even if they are Office 365 users. | Not required<br>・If the recipient is an Office 365 tenant, emails and attachments will be sent in an AIP-encrypted state.<br>・If the recipient is not an Office 365 tenant, they will be redirected to the OME (Office 365 Message Encryption) portal, where they will authenticate with a one-time passcode and then view the content in their browser.<br>・If compatible files are downloaded, they will be encrypted with AIP.<br> | Possible | ・Word (doc, docx, docm, dot, dotx dotm)<br>・Excel (xls, xlsx, xlsm, xlt, xltx, xltm, xlsb, xla, xlam)<br>・PowerPoint (ppt, pptx, pptm, pot, potx, potm, pps, ppsx, ppsm, thmx)<br>・IntoPath (xsn)<br>・XPS<br>・PDF (enablement setting required)<br>[Reference: Email How to use IRM in messages](https://support.microsoft.com/ja-jp/office/bb643d33-4a3f-4ac7-9770-fd50d95f58dc)<br>Up to 25MB including email body and attachments.<br>[Are there any limits on the size of messages that can be sent with OME?](https://docs.microsoft.com/ja-jp/microsoft-365/compliance/ome-faq?view=o365-worldwide#ome--------------------------)<br>|
| ⑤ Session control with Defender for Cloud Apps | Web access to managed SaaS apps that can be configured for authentication, such as those that are integrated with Azure AD via SSO, is routed through the Defender for Cloud Apps reverse proxy, and AIP sensitivity labels are applied or downloads are prohibited upon file download. An existing SharePoint Online site is fine, but a file sharing site accessible to external users is required. | Required | Possible | ・Word (docm, docx, dotm, dotx)<br>・Excel (xlam, xlsm, xlsx, xltx)<br>・PowerPoint (potm, potx, ppsx, ppsm, pptm, pptx)<br>・PDF<br>[Reference: Protecting files when downloading](https://docs.microsoft.com/ja-jp/defender-cloud-apps/session-policy-aad#protect-download)<br> AIP can protect files up to 50MB in size. [Applying Labels Directly to Files](https://docs.microsoft.com/en-us/defender-cloud-apps/azip-integration#how-to-integrate-azure-information-protection-with-defender-for-cloud-apps) |
<br>

# ① Anonymous Link with Password via SharePoint Online / OneDrive for Business
### Tenant Requirements
Anonymous links must be allowed in SharePoint Online / OneDrive for Business.
### Setup Overview
After uploading a file to SharePoint Online / OneDrive for Business, select Share -> "Anyone with the Link" from the file menu, set a password, uncheck "Allow Editing," and enable "Prevent Download."
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/FS_1.png">
[Reference: Sharing OneDrive Files and Folders](
https://support.microsoft.com/ja-jp/office/9fcc2f7d-de0c-4cec-93b0-a82024800c07)
<br>
<br>
# ② Send individually AIP-protected files as email attachments
### Tenant Requirements
A license including Office 365 E3 or Azure Information Protection P1 is required for AIP file protection.
### Setup Overview
Use the Office Information Rights Management feature or the Unified Label AIP Client to set permissions for external users' email addresses, generate AIP-protected files in advance, and send them as email attachments.
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/IRM2.png">
[Reference: Restricting Access to PowerPoint Presentations Using Information Rights Management](https://support.microsoft.com/ja-jp/office/bdbc7b2f-ea79-4e77-93fa-de49d48b3567)
<br>
<br>
# ③ B2B Sharing Using an IRM Library in SharePoint Online
### Tenant Requirements
- External invitations must be allowed in SharePoint Online.
- A license including Office 365 E3 or Azure Information Protection P1 is required to use the IRM library.
### Setup Overview
Target SharePointIn the library settings for your online site, enable IRM (Information Rights Management) and then upload compatible files. This setting creates a permission-protected file each time you download a file, so that only you can open it. Optional settings allow you to block the upload of incompatible files, set the file's validity period after download, and grant additional group permissions. <img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/IRM1.png">
[Reference: Applying Information Rights Management to a List or Library](https://support.microsoft.com/ja-jp/topic/3bdb5c4e-94fc-4741-b02f-4e7cc3c54aa1)
<br>
<br>
# ④Message Encryption
### Tenant Requirements
- Standard Message Encryption requires a license that includes Office 365 E3 or Azure Information Protection P1.
- Advanced Message Encryption requires a license for Microsoft 365 E5, Office 365 E5, E5 Compliant, or IP&G.
### Setup Overview
In Outlook connected to Exchange Online, when composing a new email, select "Do Not Forward" under "Encryption Options," then attach a file and send it. If the recipient is an Office 365 tenant, the email and attachments will be sent in AIP encrypted form. If the recipient is not an Office 365 tenant, they will be redirected to the OME (Office 365 Message Encryption) portal, where they can view the content in their browser after authenticating with a one-time passcode on a dedicated site. If you download a compatible file from the OME portal, it will be encrypted with AIP. <img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/FS_2.png">
[Reference: Message Encryption](https://docs.microsoft.com/ja-jp/microsoft-365/compliance/ome?view=o365-worldwide)
<br>
<br>
# ⑤ Defender for Cloud Apps
### Tenant Requirements
To use Defender for Cloud Apps, a license including Defender for Cloud Apps is required.
For Conditional Access Control, a license including Azure Active Directory P1 is required.
For file protection with AIP, a license including Azure Informatico Protection P1 is required.
### Configuration Overview
After the tenant administrator configures SSO with Azure AD, focusing on the following supported apps, session restrictions can be enabled under specific conditions using Azure AD Conditional Access. Defender for Cloud Apps settings define session policies and control the type of access using conditions such as file content and file extensions. These settings allow you to dynamically apply AIP protection to compatible files when they are downloaded under session restrictions, using sensitivity labels or permission settings limited to the downloading user. You can also block downloads while allowing viewing and editing using Office for the Web. <img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/MDCA1.png">
[Reference: Protecting files during download](https://docs.microsoft.com/ja-jp/defender-cloud-apps/session-policy-aad#protect-download)

Apps that support session control
- AWS
- Azure DevOps (Visual Studio Team Services)
- Azure portal
- Box
- Concur
- CornerStone on Demand
- DocuSign
- Dropbox
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

Apps other than those listed above may also be supported if they allow you to change the authentication method using SAML 2.0 or Open ID Connect.
[Reference: Supported Apps and Clients](https://docs.microsoft.com/ja-jp/defender-cloud-apps/proxy-intro-aad#featured-apps)ss