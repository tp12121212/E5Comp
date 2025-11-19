# Sharing files protected by Azure Information Protection with external users
Azure Information Protection allows you to share encrypted files not only with your own tenant, but also with users using external accounts such as those listed below. However, various prerequisites make it difficult for users attempting to share files to fully control them, and this feature is not widely used. This page clarifies these prerequisites and points to note to help you make a more specific decision about whether or not to use this feature.

## Test file without permissions
The following file has been encrypted and protected by granting permissions to any authenticated user (AuthenticatedUsers). Any external user should be able to open it regardless of permissions. Use this to test whether the file can be opened in your environment, regardless of permissions.
[Word file](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AIP/AnyoneProtection.docx): Requires viewing with M365 Apps or the Office + AIP UL Office add-in.
[PDF file](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AIP/AnyoneProtection.pdf): Requires viewing with Edge or AIP Viewer.
[PPNG file](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AIP/AnyoneProtection.ppng): Requires viewing with AIP Viewer.

## Accounts that can be used for AIP authentication
A. An account in an external Office 365 tenant
B. An account signed up individually with a company email address on the [Azure Information Protection](https://aka.ms/rms-signup) website.
(Email address with a consumer email domain) (Note: The use of email addresses is not possible, resulting in Azure AD Free accounts.)
C. External user accounts using an email address and one-time passcode via Azure AD B2B.
D. Consumer Microsoft Accounts.
(Microsoft Accounts can be used to open AIP-protected files in Office client apps, but cannot be used to open AIP-protected PDF files in AIP Viewer, Edge, Acrobat, etc.)

## Key Considerations
1. How to Specify External Users
2. External User Environment
3. How to Authenticate External Users Using a B2B One-Time Passcode
4. Conditional Access Considerations
5. Network Requirements for Opening AIP-Protected Files

# 1. How to Specify External Users
In Azure Information Protection, users and groups are generally specified by email address.
When opening a file, the system checks whether the specified email address matches one or more email addresses recognized by Azure Active Directory in ProxyAddresses.
If ProxyAddresses is not configured, the UserPrincipalName value is used to determine whether it matches.
[Reference: Preparing users and groups for Azure Information Protection](https://docs.microsoft.com/en-us/azure/information-protection/prepare)

Similarly, external users are specified by email address. Email-address-enabled groups (security, distribution, and Microsoft 365 groups) can also be used. You can add users from your own tenant to groups, or, if you've invited external users using Azure AD B2B, you can add their guest accounts to groups in your tenant and grant them permissions. Furthermore, if you know the email address, you can specify the email address and use it to control permissions, even for groups in external Office 365 tenants.

In addition to directly specifying users and groups as described above, it is also possible to set permissions on a domain-by-domain basis, but there are limited ways to do this, and end users must install the AIP Client, right-click on a file in File Explorer, and set permissions from the "Classify and Protect" menu. Note that if you grant permission on a domain basis, and that domain is used in an Office 365 tenant, permissions will be granted to all users in the tenant that uses that domain.
[Reference: Assign Permissions Now](https://docs.microsoft.com/ja-jp/microsoft-365/compliance/encryption-sensitivity-labels?view=o365-worldwide#assign-permissions-now)

## Summary of External User Designation Methods
| Environment | Individual User Designation<br> (including AAD group designation with valid email addresses) | Domain Designation<br> (in Azure AD, the entire tenant with that domain) | Any Authenticated User |
|:---:|:---:|:---:|:---:|
| Office 365 Apps for Business | Yes | Yes <br> (Natively supported in Microsoft 365 Apps 2303) | No |
| After installing Azure Information Client, right-click a file to set permissions | Yes | Yes | No |
| When the tenant administrator defines the sensitivity label protection settings | Yes | Yes | Yes |

## Microsoft 365 Apps -> Domain settings are now natively supported in Microsoft 365 Apps 2303.
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_PIC04.png">
(In previous versions, it was possible to specify all users in a domain using the special AllStaff-7184AB3F-CCD1-46F3-8233-3E09E9CF0E66@domain_name method.)

## After installing Azure Information Client, if you right-click a file to set permissions -> Domain settings are available.
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_PIC02.png">

## When a tenant administrator defines protection settings for sensitivity labels -> Domain settings are available.
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_PIC03.png">

If you want to encrypt files but allow any authenticated user to open them, tenant administrators can specify "Any Authenticated User" when defining the sensitivity label protection settings. In this case, since there is no control over who can open the file, you can set an expiration date to limit the file's viewing period or use tracking and expiration features in conjunction with this setting.

# 2. External User Environment
Opening AIP-protected files requires an AIP-compatible Office environment, etc., and please note that the types of accounts available vary depending on the environment and app.
[Reference: Supported scenarios for opening protected documents](https://docs.microsoft.com/ja-jp/azure/information-protection/secure-collaboration-documents#supported-scenarios-for-opening-protected-documents)

## Opening Protected Office Files
| Environment | Viewing AIP-Protected Files with Entra ID | Viewing AIP-Protected Files with Microsoft Account (Outlook.jp, Hotmail.com, etc.) | Viewing AIP-Protected Files with Entra ID B2B (including one-time passcode) |
|:---:|:---:|:---:|:---:|
| Microsoft 365 Apps | Yes | Yes | Yes |
| Office 2019 | Yes | Yes | Yes |
| Office 2013, 2016 | Yes | No | Yes |
| Office 2010 | Yes Yes, but requires AIP Client installation. | No | No. Does not support modern authentication, so cannot be used with Azure AD B2B. |
|Office for iOS (version 2.42 or later) | Yes | Yes | Yes |
|Office for Android (version 16.0.13029 or later) | Yes | Yes | Yes |
|Office for MacOS (version 16.42 or later) | Yes | Yes | Yes |

## When opening a protected PDF file
| Environment | Viewing AIP-protected files with an Office 365 account | Viewing AIP-protected files with a Microsoft Account | Viewing AIP-protected files using Azure AD B2B (including one-time passcode) |
|:---:|:---:|:---:|:---:|
|AIP Viewer for Windows/iOS/Android (supports other file formats such as ptxt and pjpg) | Yes | No | Yes |
|Edge for Windows 8, 8.1, 10, 11 (version 83.0.478.37 or later) | Yes (PDF only) | No | Yes (PDF only) |
|Acrobat / Acrobat Reader for Windows 8, 8.1, 10, 11 + [MIP plug-in for Acrobat and Acrobat Reader](https://go.microsoft.com/fwlink/?linkid=2050049) | Yes (PDF only) | No | Yes (PDF only) |
|Acrobat / Acrobat Reader for macOS 10.12 or later + [MIP plug-in foror Acrobat and Acrobat Reader](https://go.microsoft.com/fwlink/?linkid=2050049) | Yes (PDF only) | No | Yes (PDF only) |

Viewing AIP-Protected PDF Files
[Reference: Which PDF readers are supported for protected PDFs?](https://docs.microsoft.com/ja-jp/azure/information-protection/rms-client/protected-pdf-readers)
[Reference: External Users and AIP Viewers](https://docs.microsoft.com/ja-jp/azure/information-protection/known-issues#known-issues-for-the-aip-viewer)
For the Windows version of Acrobat Reader + MIP Plug-in, existing authentication information is stored in %userprofile%\AppData\Roaming\Adobe\Acrobat\DC\Security Since the files are stored in this folder, if you want to switch users or re-authenticate, please delete the files in this folder at your own risk.

# 3. External User Authentication Using B2B One-Time Passcodes
If the recipient of an AIP-protected file uses a mail server other than Exchange Online or an email account provided by a consumer Internet Service Provider, they may not have an account to open the AIP-protected file. In this case, you can use a generic one-time passcode-based Azure AD B2B invitation via a Teams team invitation or SharePoint Online external sharing feature. (You can also have users create a free Azure Active Directory account on the [Azure Information Protection](https://aka.ms/rms-signup) site, but email addresses with consumer email domains cannot be used.) To use one-time passcode-based Azure AD B2B invitations, your tenant must meet the following prerequisites:

## Prerequisites for One-Time Passcode-Based Azure AD B2B Invitations
1. One-time passcodes must be enabled in Azure AD B2B.
(If you haven't opted out, they will be enabled gradually starting November 1, 2021.)
2. To use external invitations in SharePoint Online / OneDriver for Business, the B2B integration feature must be enabled.
3. An authorized user must have already initiated a B2B external invitation.
(If external invitations are restricted in your tenant, the external user must have been invited by someone with invitation permissions, such as the Guest Inviter role.)

Once the one-time passcode-based Azure AD B2B invitation has been completed, you can use the external user's email address. By preparing and sending AIP-protected files to specified addresses or domains, you can allow users outside your company to open AIP-protected files. Note that authentication using the B2B mechanism requires the domain information of the tenant from which the file is shared, so you must also provide the domain information of the tenant from which the file will be shared to open the AIP-protected file.

## Authentication Method Using B2B/One-Time Passcode at the Shared Destination
1. When opening a protected file, select your work or school account and enter your email address once.
2. On the sign-in screen, be sure to select "Sign in Options" and "Choose to sign in to your organization."<br>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_OTP1.png"> <br>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_OTP2.png"> <br>
3. In the organization search, enter the domain name provided by the file sharer. (Usually, the domain of the sharer's email address is OK.) <br>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_OTP3.png"> <br>
5. In the following dialog, confirm that the authentication screen is branded according to the sharer's tenant environment.
6. Confirm that your email address is entered and click Next. (If you have already authenticated, you will see a screen where a code will be sent to your email address. Click "Send Code." <br>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_OTP4.png"> <br>
8. You will receive an 8-digit one-time passcode to your email address. Enter this code into the authentication screen within 30 minutes to complete the authentication. <br>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_OTP5.png"> <br>
9. If the email address or domain matches the permissions set for the shared AIP-protected file, the user can view and use the file within the scope of the permissions.

If you invite an external user through SharePoint Online with the [B2B integration feature](https://docs.microsoft.com/ja-jp/sharepoint/sharepoint-azureb2b-integration) enabled, you can also enable the [IRM](https://docs.microsoft.com/ja-jp/microsoft-365/compliance/apply-irm-to-a-list-or-library?view=o365-worldwide) feature in the SharePoint Online library and configure it to automatically protect compatible files with AIP when they are downloaded from the library.

# 4. Conditional Access Considerations
AIP While it's possible to enforce conditional access control in the sharing tenant when opening files, care must be taken when configuring AIP conditional access control for all users. Typically, if the recipients are Office 365 users with Azure Active Directory accounts in their own tenants, they can open AIP files without an explicit B2B invitation due to the implicit trust between Azure Active Directory tenants. However, if you configure AIP conditional access control for all users, external users are also subject to this conditional access control. However, it's important to be aware that if the external user does not have a B2B account in your tenant, all access to the AIP by external users will be blocked and they will not be able to pass authentication. As a result, external users without a B2B account will not be able to open AIP-protected files. Therefore, if you want to extend AIP conditional access control to external users, you must always use B2B invitations, even if they are Office 365 users or Microsoft account users. To eliminate the need for B2B invitations, you can exclude "Microsoft Rights Management Services, 00000012-0000-0000-c000-000000000000" from the Conditional Access Control policy for all users. Alternatively, if you want to configure AIP conditional access for individual users, you can configure a Conditional Access policy for specific B2B invited users or a group containing those users.
[Reference: AIP-based Conditional Access Policies](https://docs.microsoft.com/ja-jp/azure/information-protection/known-issues#aip-based-conditional-access-policies)

# 5. Network Requirements for Opening AIP-Protected Files
Opening an AIP-protected file requires connection and authentication to the cloud. Therefore, connections to the following URLs must be allowed:
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

Also, be aware of the following proxy requirements:
- Do not use an authenticated proxy, or if you are using a proxy that requires authentication, ensure that Integrated Windows Authentication is available and that the AIP registry settings are configured to use default credentials.
- Do not perform HTTPS inspection for communications with aadrm.com (do not replace the default Microsoft CA certificate).

For more information, see the following:
[Reference: Azure Information Protection Requirements](https://docs.microsoft.com/en-us/azure/information-protection/requirements#firewalls-and-network-infrastructure)