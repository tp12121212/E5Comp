# How to Apply or Exclude DLP on a Library-by-Library Basis in SharePoint Online
## Overview
When applying DLP to SharePoint Online, you typically specify the site to which DLP applies, meaning that the same DLP policy applies to all libraries within a site. However, in actual operations, there may be cases where you want to exempt DLP from application only to libraries with a high number of false positives, or you want DLP to run only on specific libraries. This article explains how to apply or exempt DLP on a library-by-library basis within a SharePoint Online site by setting the library's default retention label as a DLP condition.

## Prerequisites
### Licensing
DLP for SharePoint Online alone is covered by Office 365 E3 or SharePoint Online Plan 2. However, because we're using a default retention label for a library, users of this site will also need an Information Protection & Governance license or higher. (An AIP P1 license is also required as a prerequisite for Information Protection & Governance.)
[Reference: About default retention label licenses.](https://learn.microsoft.com/ja-jp/office365/servicedescriptions/microsoft-365-service-descriptions/microsoft-365-tenantlevel-services-licensing-guidance/microsoft-365-security-compliance-licensing-guidance#licensing-for-retention-label-policies)

### How Retention Labels Work
1. Site administrators can set one of the retention labels published for a site as the default retention label for each library.
2. When a file is uploaded to a library, the library's default retention label is applied to that file.
3. Users with general contribution permissions cannot change the library's default retention label, but can change retention labels set for files and folders.
4. When a folder is created in a library, the default retention label is not applied to the folder itself. However, if the folder does not have an individual retention policy, the default library retention label is also applied to the contents of the folder.
5. If a retention label is set for a folder in a library, the retention label set for the folder replaces the retention label for the files in the folder.

### About DLP Circumvention
Whether DLP is applied is controlled by retention labels, so DLP can be circumvented in the following ways. These methods should not be considered complete data control by DLP, but should be viewed as a means to prevent accidental errors.
1. Site administrators can change the default retention label of a library within a site by changing the library settings, thereby circumventing DLP.
3. Although DLP initially determines whether a file or folder is blocked based on the default retention label settings, a user can change the retention label of the file or folder to exclude the blocked file from DLP and unblock it.

## Setup Procedure
1. Access [Data Lifecycle Management](https://compliance.microsoft.com/informationgovernance) in the Microsoft Purview portal.
1.A. If you want to exclude a specific library from DLP, create a retention label with a name such as "Shareable" without any retention settings.
1.B. If you want to apply DLP only to a specific library, create a retention label with a name such as "Cannot Share" without any retention settings. <img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/DLPbyRetention01.png"/>

2. Publish the retention label created in step 1 to a specific SharePoint Online site using Data Lifecycle Management's [Label Policies](https://compliance.microsoft.com/informationgovernance?viewid=labelpolicies).

3. After step 2, wait a few hours, then access the library in the SharePoint Online site in question. From the gear menu in the upper right, go to Library Settings -> Other Library Settings and select "Apply a label to items in this list or library."
3.A. If you want to exclude this library from DLP, set the "Shareable" retention label as the default retention label.
3.B. If you want to apply DLP to only this library, set the "Not Shareable" retention label as the default retention label. <img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/DLPbyRetention02.png"/>

4. Create a DLP policy for the specific SharePoint Online site listed above in the Microsoft Purview portal under [Data Loss Prevention](https://compliance.microsoft.com/datalossprevention?viewid=policies).
4.A. To exclude specific libraries from DLP in a DLP rule, add a "Group," select "Retention Label" from "Add Condition," and add the "Shareable" label. Enable "Not" for the rule group and set it to exclude items with the "Shareable" retention label. Set other DLP conditions as appropriate. <img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/DLPbyRetention03.png"/>
4.B. To apply DLP to only specific libraries, select "Retention Labels" from "Add Conditions," add the "Do Not Share" label, and set the target to only items with the "Do Not Share" retention label. Set other DLP conditions as appropriate. <img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/DLPbyRetention04.png"/>

## Search by Retention Label
To search for only items in a library with a specific retention label, you can use the managed property "ComplianceTag" and search with the following KQL:
`ComplianceTag:"Cannot be shared"`

To search for only items in a library that do not have a specific retention label, use the managed property "ComplianceTag" and the following KQL:
`-ComplianceTag:"Can be shared"`