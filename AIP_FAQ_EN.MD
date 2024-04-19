# Share files protected by Azure Information Protection with external users

Azure Information Protection allows you to share encrypted files not only with your own tenants, but also with users who use the following external accounts. However, it is difficult to completely control from users who want to share files, and there are various prerequisites, so it seems that the utilization is not progressing much. On this page, we will clarify these prerequisites and precautions for the purpose of making a more specific decision on availability.

## Test files that do not depend on authority

The following files are encrypted and protected by granting permissions to any authenticated users (AuthenticatedUsers), and any external user should be able to open it regardless of authority. Please use it to test whether it is an environment to open files without permission.

[Word file](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AIP/AnyoneProtection.docx): M365 A You need to refer to pps or Office + AIP UL Office add-in

[PDF file](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AIP/AnyoneProtection.pdf): Edge or Needs to be referenced in AIP Viewer

[PPNG file](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AIP/AnyoneProtection.ppng): AIP Viewer Reference is required

## Accounts that can be used for authentication with AIP
A. External Office 365 tenant account<br>
B. Accounts that signed up individually using the company's email address on the [Azure Information Protection](https://aka.ms/rms-signup) site<br>
(It is not possible to use an email address with a consumer email domain, and as a result, it becomes an Azure AD Free account.)<br>
C. External user account using email address using Azure AD B2B mechanism + one-time passcode<br>
D. Consumer Microsoft Account<br>
(Microsoft Account is available when opening AIP-protected files in Office client apps, but<br>AIP Viewer, Edge, Acrobat, etc. cannot use Microsoft Account when opening AIP-protected PDF files, etc.)

## Main considerations
1. How to specify an external user
2. External user environment
3. Authentication method for external users using B2B one-time passcode
4. Conditional access considerations
5. Network requirements for opening AIP protected files

# 1. How to specify an external user
Basically, Azure Information Protection specifies users and groups by email address.

Also, when opening the file, a single or multiple email address set in ProxyAddresses recognized by Azure Active Dreictory, or

If there is no setting in ProxyAddresses, the value of UserPrincipalName determines whether it matches the specified email address.

[Reference: Preparing users and groups for Azure Information Protection](https://docs.microsoft.com/en-us/aZure/information-protection/prepare)

Similarly, external users are also specified by email address. In addition, groups with valid e-mail addresses (security, distribution groups, Microsoft 365 groups) can also be used to add users of their own tenants to the group, and Azure AD B2B If you invite an external user with the function of , you can also add the guest account to your own tenant group and grant permission. Also, if you know the e-mail address, even if it is a group of Office 365 tenants outside the company, you can specify the e-mail address and use it for permission control.

In addition to specifying direct users and groups like the above, it is also possible to set permissions on a per-domain basis, but the methods that can be set are limited, and for end users, install AIP Client and exp It is important to note that you need to right-click on the file on the roller and set permissions from the "Classify and Protect" menu. If it is allowed by the domain, if it is a domain used by the Office 365 tenant, the user of the entire tenant using the domain will be given authority.

[Reference: Assign permissions now](https://docs.microsoft.com/ja-jp/microsoft-365/compliance/encryption-sensitiviTy-labels?View=o365-worldwide#assign-permissions-now)

## Summary of how to specify external users
| Environment | Individual user designation <br> (including group designation of AAD with valid email address) | Domain specification <br> (In Azure AD, the entire tenant with that domain) | Authenticated user - Anyone |
|:---:|:---:|:---:|:---:|
|Office 365 Apps for Business | 〇 | 〇 <br> (Native support on Microsoft 365 Apps 2303) | × |
|After installing Azure Information Clienet, right-click on the file and set permissions | 〇 | 〇 | × |
| When the tenant administrator defines the protection setting of the confidentiality label | 〇 | 〇 | 〇 |

## Microsoft 365 Apps -> Domain settings are also natively supported by Microsoft 365 Apps 2303
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_PIC04.png">

(Even in previous versions, it is possible to specify all users of the domain with a special AllStaff-7184AB3F-CCD1-46F3-8233-3E09E9CF0E66@ domain name.)

## After installing Azure Information Clienet, right-click on the file to set permissions -> You can set the domain
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_PIC02.png">

## When the tenant administrator defines the protection setting of the confidentiality label -> You can set the domain
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_PIC03.png">

In addition, although the file is encrypted, if any authenticated user can open it, the tenant administrator can also specify "any authenticated user" when defining the protection setting of the confidentiality label. In this case, we do not control who can open the file at all, so it is possible to set the expiration date and limit the reference period of the file, and use it together with the tracking and expiration functions.

# 2. External user environment
When opening AIP protected files, you need an Office environment that supports AIP, etc., and you need to be careful that the types of accounts available vary depending on the environment and app.

[Reference: Scenarios supported when opening protected documents](https://docs.microsoft.com/en-jp/azure/information-protection/secure-collaboration-documents#supported-scenarios-for-opening-protected-documents)

## When opening a protected Office file
| Environment | Reference AIP-protected files in Office 365 account | Reference AIP-protected files in Microsoft Account | Azure AD B2B (Wa Reference of AIP protected files using time (including time passcode) |
|:---:|:---:|:---:|:---:|
|Microsoft 365 Apps | 〇 | 〇 | 〇 |
|Office 2019 | 〇 | 〇 | 〇 | 〇 |
|Office 2013, 2016 | 〇 | × | 〇 |
|Office 2010 | 〇 You can open it, but you need to install AIP Client | × | × It does not support advanced authentication, so it can not be used with Azure AD B2B |
| Office for iOS (version 2.42 or later) | 〇 | 〇 | 〇 |
| Office for Android (version 16.0.13029 or later) | 〇 | 〇 | 〇 | 〇 |
| Office for MacOS (version 16.42 or later) | 〇 | 〇 | 〇 | 〇 |

## When opening a protected PDF file
| Environment | Reference AIP-protected files in Office 365 account | Reference AIP-protected files in Microsoft Account | Azure AD B2B (Wa Reference of AIP protected files using time (including time passcode) |
|:---:|:---:|:---:|:---:|
| AIP Viewer for Windows/iOS/Android (also supports other file formats such as ptxt, pjpg, etc.) | 〇 | × | 〇 |
| Windows 8, 8.1, 10, 11 version Edge (version 83.0.478.37 or later) | 〇 (PDF only) | × | 〇 (PDF only) |
|Windows 8, 8.1, 10, 11 edition Acrobat / Acrobat Reader + [MIP plug-in for Acrobat and Acrobat Reader](htt Ps://go.microsoft.com/fwlink/? Linkid=2050049) | 〇 (PDF only) | × | 〇 (PDF only) |
| Acrobat / Acrobat Reader + [MIP plug-in for Acrobat and Acrobat Reader] on macOS 10.12 or later (https://go.microsoft.com/fwlink/?Linkid=2050049) | 〇 (PDF only) | × | 〇 (PDF only) |

AIP protected PDF file reference

[Reference: Which PDF reader is supported for protected PDFs?](Https://docs.microsoft.com/ja-jp/azure/information-protection/rms-client/protected-pdf-readers)

[Reference: External users and AIP viewers](https://docs.microsoft.com/ja-jp/azure/information-protection/known-issues#known-issues-for-the-aip-viewer)

In the case of Acrobat Reader + MIP Plug-in for Windows, the existing authentication information is %userprofile%\AppData\Roaming\Adobe\Acrobat\ Since it is stored in DC\Security, if you want to switch users or redo authentication, delete the files in this folder at your own risk.

# 3. Authentication method for external users using B2B one-time passcode
The destination where the AIP-protected file is shared is using a non-Exchange Online mail server or provided by various consumer Internet service providers. If you are using a count, you do not have an account to open AIP-protected files. In this case, you can respond with a general-purpose one-time passcode-based Azure AD B2B invitation using Teams' team invitation or the external sharing function of SharePoint Online. It is. ( In addition, on the [Azure Information Protection](https://aka.ms/rms-signup) site, Azure Active Directory's Free It is also possible to create a count, but it is not possible to use an email address with an email domain for consumers.) When using a one-time passcode-based Azure AD B2B invitation, the tenant must meet the following prerequisites.

## Prerequisites for one-time passcode-based Azure AD B2B invitations

1. [One-time passcode] in B2B in Azure AD (https://docs.microsoft.com/ja-jp/azure/active-directory/external-ident Ities/one-time-passcode) is enabled

(If you do not opt out, it will be activated sequentially from November 1, 2021 or later)

2. If you want to externally invite in SharePoint Online / OneDriver for Business, [B2B integration function] (https://docs.microsoft.com/ja-jp/ Sharepoint/sharepoint-azureb2b-integration) is enabled

3. An authorized user must implement a B2B external invitation once.

(If the external invitation is restricted by the tenant, the target external user is B2B invited by the operation of the person who has the invitation authority, etc. in the role of the Guest Inviter, etc.

If you have an Azure AD B2B invitation based on the above one-time passcode, you can prepare and send an AIP-protected file by specifying the email address and domain of users outside the company, and users outside the company You can also open AIP-protected files. In addition, when authenticating using the B2B mechanism, the domain information of the shared tenant is required, so **AIP also convey the domain information of the shared tenant used when opening the protected file. You need to put it**.

## Authentication method using B2B/one-time passcode at the shared destination

1. When opening a protected file, select your work or school account and enter your own email address once

2. On the sign-in screen, be sure to select "Sign in Options" and "**Choose sign in to your organization**"<br>

<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_OTP1.png"> <br>

<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_OTP2.png"> <br>

3. In the organization search, **Enter the domain name presented by the file sharing source **<br> (usually OK with the domain of the shared email address) <br>

<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_OTP3.png"> <br>

5. In the following dialog, confirm that the authentication screen where the brand of the shared tenant environment is applied will be displayed.

6. Confirm that your email address has been entered and click Next <br> (Once authenticated, you will be on the screen to send the code to your email address, so click Send "Code" <br>

<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_OTP4.png"> <br>

8. You will receive an 8-digit one-time passcode to your email address, so enter the code on the authentication screen within 30 minutes to complete the authentication <br>

<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_OTP5.png"> <br>

9. If it matches the authorized email address or domain in the shared AIP protection file, you can refer to and use the file within the scope of the permission setting.

Through SharePoint Online [B2B integration function] (https://docs.microsoft.com/ja-jp/sharepoint/sharepoint-azureb2b If you want to externalite with -integration) enabled, in the SharePoint Online library, [IRM] (https://docs.microsoft.com/ja-jp/m Icrosoft-365/compliance/apply-irm-to-a-list-or-library? View=o365-worldwide) When you enable the function and download the corresponding file from the library, you can also take the configuration that it is automatically protected by AIP.

# 4. Conditional access considerations

When opening AIP files, it is possible to control conditional access at the shared source tenant, but you need to be careful when setting up AIP conditional access control for all users. Normally, if you are an Office 365 user and your tenant has an Azure Active Directory account, Azure Activit Directory With implicit trust between tenants, it is possible to have the AIP file opened without a particularly explicit B2B invitation. However, **If you set up AIP conditional access control for all users**, external users are no exception and are subject to this conditional access control, but as a constraint to be recognized, **in-house tenant side If the B2B account of the applicable external user does not exist**, all access to the AIP by the external user will be blocked and cannot be authenticated, and as a result **B2B account exists Outside users who do not have it will not be able to open AIP-protected files**. Therefore, if you want to deploy AIP conditional access control including external users, even if the other party is an Office 365 user or a Microsoft account user, you will always invite B2B. It's necessary to keep it. On the other hand, if you want to eliminate the need for B2B invitation, if you want to exclude AIP from the conditional access control policy for all users, or if you want to set AIP conditional access individually, specific Set up a conditional access policy only for B2B invited users and groups that include those users.

[Reference: AIP-based conditional access policy] (https://docs.microsoft.com/en-jp/azure/information-protection/known-issu Es#aip-based-conditional-access-policies)

# 5. Network requirements for opening AIP protected files

When opening an AIP-protected file, a connection to the cloud and authentication occur. Therefore, the connection to the following URL must be allowed.

- login.microsoftonline.com

- \*.aadrm.com

- \*.azurerms.com

- \*.informationprotection.azure.com

- \*.cloudapp.net

- ecn.dev.virtualearth.net

- informationprotection.hosting.portal.azure.net

- \*.protection.outlook.com

- config.edge.skype.com

- \*.events.data.microsoft.com

- \*.aria.microsoft.com (Android devices only)

You should also pay attention to the following proxy requirements.

- Even if you are not using an authenticated proxy or are using a proxy that requires authentication, you can use integrated Windows authentication and set the AIP registry to use the default authentication information. To be

- Not doing HTTPS inspection in communication with aadrm.com (not HTTPS communication replacing the default Micorsoft CA certificate)

For details, please check the following.

[Reference: Azure Information Protection requirements] (https://docs.microsoft.com/ja-jp/azure/information-protection/Requirements#firewalls-and-network-infrastructure)