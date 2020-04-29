---
layout: post
title:  "Azure AD Administrative Units - Use cases, considerations and limitations"
author: thomas
categories: [ Azure, Security, AzureAD ]
tags: [azure, azuread, security]
image: assets/images/adminunits.jpg
description: "Administrative Units (AUs) allow organizations to delegate admin permission to a custom scope and segment (such as region, department, business units) within a single Azure AD tenant. In this blog post I like to share my experience including use cases, considerations and limitations of the AU management (preview) feature"
featured: false
hidden: false
---

_Administrative Units (AUs) allow organizations to delegate admin permission to a custom segment of a tenant (such as region, department, business units). In this blog post I like to share my experience including use cases, considerations and limitations of the AU management (preview) feature._
 
## What are “Administrative Units” (AUs)?
In general, Azure AD has a flat structure where objects are located on the same “level”. In the (old) days of Active Directory the management “Organizational Units” (OUs) helps to delegate administration and apply policies (GPO) based on a designed structured. This is why some initial thought is that AUs are an 1:1 adoption of the traditional OUs concept in Azure AD. On second glace it will be clear that this concepts has some major differences (as described in this blog post).

However, you may looking for a solution to segment users or groups for delegation of permissions on a self-named structure (based on container) now or at any future time.

Administrative Units are available as public preview since [December 2014](https://azure.microsoft.com/en-us/updates/public-preview-administrative-units/) (!).
But the management of AUs was limited to Graph API and [PowerShell](https://docs.microsoft.com/en-us/powershell/azure/active-directory/working-with-administrative-units?view=azureadps-2.0).  In the latest preview release (April 2020), Microsoft introduce the option to using the Azure Portal. 

This is a good time to check how this feature can support you and what are the considerations or limitations.

## General purposed use of AUs
Global and Privileged Role Admins are able to manage AUs in an Azure AD tenant.
These containers are purposed for management and delegation tasks only.
It is obvious that Microsoft has decided to separate the authorization model
(such as using Security Groups) from a “resource grouping” approach (in this case with AUs).

This model should assists organizations to assign least-privileges on a minimum scope and provide an option to build a (logical) management structure for resources within a single tenant.
It supports also large or multi-site organizations to bring structure to the tenant. 😉

## Characteristics of AUs
* Only certain types of resources can be assigned
(so far restricted to users and groups)
* Relationship can be established in one-to-many (one resource assigned to multiple AUs)
	* No one-to-one relation or unique allocation neccessary
(strong difference to Organizational Units in AD)
* No hierarchical structure, nesting and inheritance
(in the style of “Sub-AUs” 😀) 
* Membership to AU with single assignment or bulk import
(no dynamic assignment support)
* Azure AD (RBAC) Directory Roles can be assigned on scope of AUs
(roles has to support this as well, currently very limited)
* Management via PowerShell, Graph API and Portal UI

## Management of AUs
Microsoft has written a very good documentation on how to manage AUs and assignment of directory roles.
Saying this, I like to reference on the related official Microsoft Docs:

* [Manage administrative units](https://docs.microsoft.com/en-us/azure/active-directory/users-groups-roles/roles-admin-units-manage) with Azure Portal and PowerShell
* Add and manage [users](https://docs.microsoft.com/en-us/azure/active-directory/users-groups-roles/roles-admin-units-add-manage-users) and [groups](https://docs.microsoft.com/en-us/azure/active-directory/users-groups-roles/roles-admin-units-add-manage-groups) in an AU
* [Assign scoped roles](https://docs.microsoft.com/en-us/azure/active-directory/users-groups-roles/roles-admin-units-assign-roles) to an AU

## Real-world use cases and scenarios of AUs
In the past I’ve faced the following limitations or questions around the existing flat structure of Azure AD:

* Delegated permission on geographical, organizational or intra-company units
	* Example: Local helpdesk should be able to manage users and groups of local office only
* Segmentation of privileged (sensitive) accounts and work (hybrid) accounts
	* Example: User or Password Administrator are able to reset passwords for “non-administrator” only. Microsoft already [mentioned in the documentation](https://docs.microsoft.com/en-us/azure/active-directory/users-groups-roles/directory-assign-admin-roles#user-administrator) that subscription owners (Azure resources) or privileged accounts outside of Azure AD RBAC/Directory Roles (such as Intune- or MDATP-RBAC) will be included as “non-administrators”. 
	
In my opinion these are good examples where AUs can supports to limit the scope.

## Visibility and usage of AUs in Portals
### My Staff portal
This portal (mystaff.microsoft.com) seems to be a great option for administrators and helpdesk employees to manage users on a certain (AU) scope.
It is also optimized for touch based input and the UI is sharply reduced (compared to Azure AD portal). 
Therefore, it could be also interesting for 1st-level helpdesk, on-call service or mobile admins.

Access to the “MyStaff” portal can be restricted by the user feature preview settings in the Azure portal:

![](../2020-04-29-azuread-administrative-units/mystaff-settings.png)

Delegated admins are able to see the assigned AU only:

![](../2020-04-29-azuread-administrative-units/mystaff-au.png)

List of all users in the AU will be displayed:

![](../2020-04-29-azuread-administrative-units/mystaff-allusers.png)

Password Admins are able to reset user's password in the reduced UI. Option to “Add phone number” is grayed-out because of the missing directory role assignment of “Authenticator Admin”:

![](../2020-04-29-azuread-administrative-units/mystaff-user.png)


### Microsoft 365 Admin Portal
Users with assigned directory roles of an AU-level are also able to use the Microsoft 365 Portal.
They will find a drop down in the left corner for showing them the current and assigned AU scope.
Only resources from this AU will be displayed in the portal:

![](../2020-04-29-azuread-administrative-units/m365portal.png)

### Azure Audit and sign-in logs
Currently the Azure AD Audit Log schema contains already an “administrativeUnits” column of the TargetRessources entry. But has not been filled or used until now. So it seems that Microsoft is planning to include the information of AU assignment(s):

![](../2020-04-29-azuread-administrative-units/azureadauditlogs.png)

### Role Assignments in the Azure Portal
Assigned directory roles on level of AUs are not visible in (tenant-level) Azure Portal overview
(outside of the AU management):


Directory Role Assignment in "**Azure AD Roles and Administrators**":
![](../2020-04-29-azuread-administrative-units/roles-tenant-overview.png)



Display of "Your Role" also not showing AU-assigned permissions:
![](../2020-04-29-azuread-administrative-units/roles-myassign.png)



Directory Role Assignment in "**Administrative Unit Management**":
![](../2020-04-29-azuread-administrative-units/roles-tenat-auview.png)

All of this can be confusing because of a missing overall view of directory role assignments in the portal. 

## Missing support and potentially side effects of Azure AD PIM
What I was missing the most during my tests was the eligible assignments in Azure AD PIM.
I hope this will be added very soon.
You should take care that **AUs are not supported in PIM**.
Even if it seems that you have the option to convert an AU-scoped role assignment in Azure AD PIM!


In contrast of the Azure AD blade, PIM shows the assignment of “Password Administrator”.
But unfortunately PIM displays the scope as “Directory” (Tenant-wide) and not as configured (in the scope of AU "Hybrid Identities"):

![](../2020-04-29-azuread-administrative-units/pim-overview.png)

This scope is shown incorrect because of the missing support of AUs as a configurable scope in Azure AD PIM.
Please be aware that converting this role from permanent to eligible assignment will also set the focus to "Directory".
Your previous AU-scoped assignment will be overwritten.

## Conclusion: *Current* limitations and considerations of AUs
Administrative Units are a great way to build a flexible structure for delegation and management.
From my point of view there are already use cases where it covers all requirements.
But keep in mind and consider the following limitations (at the time of writing this blog post): 

* Currently assignments of resources is limited to users and groups
(no support for devices, service principals,…)
* Using of automation or scripts is the only option to have a initially or dynamic assignment yet
* Consider the [supported scenarios](https://docs.microsoft.com/en-us/azure/active-directory/users-groups-roles/directory-administrative-units#administrative-unit-management) of using Graph API, Azure and M365 Portal for AU management
* Be aware that users can be assigned to one or more AUs
* Only a few [Directory Roles are available](https://docs.microsoft.com/en-us/azure/active-directory/users-groups-roles/roles-admin-units-assign-roles#roles-available) for assignment to AUs yet
* Azure AD PIM is currently not supported, Permanent assignment can be used only
* This feature needs Azure AD P1 (or higher) for admins that are using AUs
	* Therefore it seems not supported in B2C tenants











<span style="color:silver;font-style:italic;font-size:small">Original cover image by [geralt / Pixabay](https://pixabay.com/illustrations/social-media-personal-909708/)</span>
