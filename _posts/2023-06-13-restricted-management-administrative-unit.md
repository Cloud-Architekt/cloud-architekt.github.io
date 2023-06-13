---
title: "Protection of privileged users and groups by Azure AD Restricted Management Administrative Units"
excerpt: "Restricted Management Administrative Unit (RMAU) allows to protect objects from modification by Azure AD role members on directory-level scope. Management permissions will be restricted to granted Azure AD roles on scope of the particular RMAU. In this blog post, we will have a look on this feature and how you can automate management of RMAUs with Microsoft Graph API. In addition, I will explain use cases, limitations and why this feature support to implement a tiered administration model."
header:
  overlay_image: /assets/images/2023-06-13-administrative-units-restricted-management/rmau3.png
  overlay_filter: rgba(102, 102, 153, 0.85)
  teaser: /assets/images/2023-06-13-administrative-units-restricted-management/rmau3.png
search: true
toc: true
toc_sticky: true
categories:
  - Azure AD
tags:
  - AzureAD
  - PrivilegedIAM
  - Azure
last_modified_at: 2023-06-13
---

# Protection of Privileged Users and Groups by Azure AD Restricted Management Administrative Units

*Protection of privileged users and groups outside of Azure AD roles needs particular care to prevent privileged escalation because those objects are not protected by default in Azure AD. For example, service-specific Azure AD roles (e.g. Intune or Windows 365 Administrator) has been able to modify security groups with assigned privileges in Azure RBAC or any other non-Azure AD RBAC.*

*In this blog post I like to describe and explain the new option for "Restricted Management Administrative Units" (RMAUs) which allows to restrict management of assigned objects from Azure AD role members on tenant-level. Permissions on assigned resources in the RMAU will be restricted to the administrators scoped on the level of an Administrative Unit (AU). A focus will be also set to automated management of RMAUs via Microsoft Graph API. In addition, I will explain use cases and why this feature becomes an important part to implement a tiered administration model ("[Enterprise Access Model](https://learn.microsoft.com/en-us/security/privileged-access-workstations/privileged-access-access-model)") but also which scenarios are unsupported.*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-06-13-administrative-units-restricted-management/rmau17.png)

*Azure Active Directory has a flat hierarchy by default. In the past, Administrative Units already allowed to scope some directory roles on "Administrative Units" instead of the "Directory" (tenant-level) scope. Nevertheless, Azure AD role assignment on tenant-level has been inherited to all objects in AUs.* 

## Overview of Restricted Management Administrative Units (RMAU)

### Why use Restricted Management AU?

In the past, privileged users and groups have been protected (by default) as far they are assigned to an Azure AD admin role or role-assignable group. Furthermore, the management to those group members has been restricted to "Global Admin" and "Privileged Authentication Administrator" during a permanent or active assignment (in case of using Azure PIM for Groups) to role-assignable groups. But also permissions to manage groups and memberships have been restricted to "Global Admins", "Privileged Role Administrator" and Group Owners.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-06-13-administrative-units-restricted-management/rmau1.png)

*Overview of restricted management on users and groups to "Global and Privileged Authentication Administrators" by using different group types and PIM features in Azure AD.*

_Side Note: A limit of [500 role-assignable groups exist on tenant-level](https://learn.microsoft.com/en-us/azure/active-directory/roles/groups-concept#restrictions-for-role-assignable-groups) and management of protected users can not delegated to custom or scoped Azure AD admin roles. Therefore, it makes sense to use this group type for certain scenarios only. From my point of view, protection of users and groups by RMAU is also interesting to avoid reaching this limit._

Protecting users and groups on "[Management and Data/Workload plane](https://docs.microsoft.com/en-us/security/compass/privileged-access-access-model#tier-1-splits)" (outside of Azure AD directory roles) has been a challenge in the past. Members with assigned [Microsoft 365 service-specific directory roles](https://docs.microsoft.com/en-us/azure/active-directory/roles/concept-understand-roles#categories-of-azure-ad-roles) (such as [Intune](https://docs.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#intune-administrator), Knowledge or [Windows 365 administrators](https://docs.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#windows-365-administrator)) has been able to modify security group objects on tenant-level. These permission needs to be restricted to avoid privileged escalation. For example, gaining access to Azure resources by modifying Azure RBAC-assigned security groups

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-06-13-administrative-units-restricted-management/rmau2.png)

*Security group will be mostly used on "Management and Workload/Data plane" (such as Azure DevOps, Azure resources or Microsoft 365 RBAC systems). This includes the risk of manipulation by privileged identities with "Group Management" permissions. Helpdesk, User Administrators or other similar roles has been able to "take over" accounts with privileges outside of Azure AD. Limitation on fine-grained scoping or custom directory roles has been a "blocker" to avoid privilege escalation paths in the past.*

Restricted Management Administrative Units (RMAU) allows to restrict management of assigned users and groups by Azure AD role assignments on "Directory" scope. For example, "User Administrators" will not be able to change password of RMAU assigned accounts (for example, CEO, Developers) by default. An administrator with assigned "Group Administrator" role can be prevented from changing membership of security groups if they are assigned to a RMAU.

**In summary, you need an explicitly assigned permission to modify objects which are assigned to an RMAU. Keep in mind, this means that additional assigned directory roles with scope on RMAU-level are needed if management permissions by specific workflows and administrators  shall be maintained.**

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-06-13-administrative-units-restricted-management/rmau3.png)

_Restricted AUs block inheritance of Azure AD roles on directory roles, particular role assignments on AU level is needed to manage assigned objects. Keep in mind, only a limited set of built-in roles are supported for RMAU-level role assignments. Custom roles are supported and gives you advanced capabilities for limit permission set on least privilege principle._

### Supported objects and other limitations
Supported object types for Restricted AUs are users, security groups and devices. Unfortunately, security groups which are managed in Azure AD PIM ("PIM for Groups") as well as mail-enabled and Microsoft 365 Groups aren’t not supported.

An overview about the [supported objects types](https://learn.microsoft.com/en-us/azure/active-directory/roles/admin-units-restricted-management#what-objects-can-be-members) are documented in Microsoft Learn.

RMAUs do not cover an restriction to manage those objects as group members outside of the restricted management. This includes also protection for RMAU-assigned objects outside of the Azure AD management (e.g., device objects in Intune or mailbox settings in Exchange). Keep this in mind for other scenarios, e.g. protecting devices from assignments to device policies in Intune. A full overview of [supported operations](https://learn.microsoft.com/en-us/azure/active-directory/roles/admin-units-restricted-management#what-types-of-operations-are-blocked) are listed in Microsoft Learn.



## Which types and scope of administrative privileges are restricted?

### Directory Roles
By default, inherited permissions of (tenant-level) directory roles will no longer been applied to users, devices and groups in RMAUs. This includes also "Global Administrator" (GA), "Privileged Role Administrator" (PRA) and "Privileged Authentication Administrator" (PAA). Delegation to manage objects in RMAUs can be achieved by using respective roles on the scope of the specific Administrative Unit. Nevertheless, Global and Privileged Role Administrators can modify role assignments with scope on RMAUs. So, they’re able to grant self-assigned permissions to RMAUs.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-06-13-administrative-units-restricted-management/rmau4.png)

_By design, Global Admins are blocked from managing objects which have been assigned to RMAU. An assignment as "User Administrator" in scope of the certain RMAU is needed to manage objects from a RMAU._

**Important note**: _There’s a conflict for managing objects which are assigned to RMAU but also protected by Azure AD privileged assignments (such as active assignment to role-assignable groups or active/eligible assignment to Azure AD admin roles). Those objects are already restricted to be managed by GA and PRA only. If you add those objects to an RMAU, the inheritance will be disable and there’s no option to assign GA or PRA role particular on RMAU-Level. Therefore, make sure that protected objects by Azure AD roles and role-assignable groups will not be also protected by RMAU. Otherwise, you have no option to manage them until the RMAU assignment will be removed._

### Owner of Role-Assignable Groups (with/without enabled PIM for Groups)

Delegation to manage role-assignable groups as permanent or eligible "Owner" will be restricted when the groups are member of a RMAU. Assigning RMAU-scoped permissions as "Group Administrators" will also not allow to manage this group. As already mentioned in the side note and described in the previous table, this type of objects will be already protected by another built-in protection layer in Azure AD. You should consider managing these objects outside of RMAU and ask yourself if additional protection by RMAU is really needed.

Microsoft has documented this behavior as [limitation of RMAUs](https://learn.microsoft.com/en-us/azure/active-directory/roles/admin-units-restricted-management#limitations).

### Owner of Security Groups (with/without enabled PIM for Groups)

Delegation to manage security groups by any tenant-level Azure AD admin role but also as permanent or eligible "Owner" will be restricted when these objects are member of a RMAU. If you want to delegate management permissions to this group, assignment of "Group Administrators" needs to be scoped on RMAU. Nevertheless, "PIM for Groups" is not supported in combination with RMAU.

###  Microsoft Graph API Permissions

Service Principals with `AdministrativeUnit.ReadWrite.All`  permissions are able to add or remove members of RMAU. But the service principals will no longer been able to manage objects after the object has been added to RMAU. Permissions such as `User.WriteRead.All` or `Groups.ReadWrite.All` will be also restricted (similar to role assignments of "User or Group Administrator" on tenant-level). You will receive the following error message:

>Insufficient privileges to complete the operation. Target object is a member of a restricted management administrative unit and can only be modified by administrators scoped to that administrative unit. Check that you are assigned a role that has permission to perform the operation for this restricted management administrative unit. Learn more: https://go.microsoft.com/fwlink/?linkid=2197831

Currently you can grant permission to manage objects from a RMAU by assigning the API Permission `Directory.Write.Restricted` which establishes the access on a tenant-level scope. Verify if an assigned RMAU-scoped directory roles to the service principals would be a better solution in aspects of a "least privilege" approach.

But keep in mind, removing objects from the RMAU to use management permissions have always been possible (as part of the `AdministrativeUnit.ReadWrite.All` permission).

### Other privileged access paths

During my tests, I’ve tried some privileged access paths outside of Azure AD role members or Microsoft Graph API permissions. This has been also potential privileged escalation paths in the past.

- Users assigned to RAMU are not protected from "account takeover" by Azure AD Connect synchronization (even if soft- and hard match are not blocked). Check out the "[Azure AD Attack & Defense](https://github.com/Cloud-Architekt/AzureAD-Attack-Defense/blob/main/AADCSyncServiceAccount.md#attack-scenarios)" chapter about abusing synchronization service accounts if you are interested to learn more about soft- and hard match.
- Assignment of membership to groups in RMAUs by using "Access Packages" in Identity Governance continued to function as expected. Keep in mind, delegated administrators ("Access package assignment manager" or "Access package manger") will be able [to assign or create access packages](https://twitter.com/Thomas_Live/status/1481538197930889218) in Entitlement Management which includes groups from RMAUs.

## Overview of protection and delegation capabilities by using RMAU and role assignable groups
As already described in detail, a couple of scenarios using role-assignable groups or "PIM for Groups" are not supported or suitable in combination with RMAU. The following table should help to keep an overview about restriction of members as assignment of RMAU but also in relation to role assignable group.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-06-13-administrative-units-restricted-management/rmau_overview.png)

_Side Note: All data without warranty! I will continue some additional tests to double check the results because of the huge scope of scenarios and combinations. Feedback is always welcome!_

From my point of view, the following use cases (as an example) could be evaluated to decide if role-assignable groups or protection by RMAU (based on the previous named considerations) are usable for you:

* Use "role assignable groups" for Azure AD roles and protecting high-privileged users (on "Control Plane/Tier0" in Azure AD or high-sensitive permissions on "Management Plane/Tier1" or Cloud Platform) but avoid assignments for those group types and assigned members to RMAU.
* Assign privileged or sensitive security groups which are not managed in Azure AD PIM ("PIM for Groups") and should be restricted from Azure AD admins with "Group Management" permissions (e.g., sensitive configuration or provisioning groups). This includes also groups with eligible assignments to Azure (Resources) PIM.
* Assign privileged or sensitive users (for example: C-Level accounts, DevOps) without protection by role-assignable groups or Azure AD role assignmenet to RMAU for establishing a restricted management (marked above "Unrestricted*"). This can also cover eligible member of "PIM for Groups".
* Assign privileged or sensitive device objects to RMAU to protect extension attribute (e.g., used in Device Filters).

In the following descriptions and use cases, I will set my focus to restrict and isolate users or groups which will be used in Azure-related RBAC systems (security groups with eligible assignment to Azure Resources). RMAU provides restriction capabilities to establish a tiered administration as part of the "[Enterprise Administration Mode](https://docs.microsoft.com/en-us/security/compass/privileged-access-access-model)". The management of privileged objects in RMAUs should be restricted to "Control plane" (Tier0) admins (by Global Admins or dedicated admins with RMAU-scoped assignments) or integrated to secured identity governance processes (e.g., Entra ID Governance) only.

## Manage RMAU via Microsoft Graph API

In the next section I would like to give some code samples about managing RMAU with PowerShell and Microsoft Graph API.

### Create and manage RMAUs by Microsoft Graph API

The cmdlet `Invoke-MgGraphRequest` from the Microsoft Graph SDK will be used in the first sample to call the API endpoint. But there are also cmdlets to manage AUs directly, as you can see in the other samples as well. I’m using a Service Principal which needs AdministrativeUnit.ReadWrite.All" but no other or special permissions for creating RMAUs.

```powershell
$Body = '
{
    "displayName": "Tier1-ManagementPlane.Azure",
    "description": "This administrative unit contains assets of ManagementPlane in Azure",
    "isMemberManagementRestricted": true
}'

Invoke-MgGraphRequest -Method "POST" -Body $Body -Uri https://graph.microsoft.com/beta/administrativeUnits
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-06-13-administrative-units-restricted-management/rmau5.png)

*Properties of RMAU in the Azure Portal after creation via Microsoft Graph API*

### Assignment of objects to RMAU

Assignment of objects to a restricted management AU requires `AdministrativeUnit.ReadWrite.All` permission only. In addition, `User.Read.All` and/or `Group.Read.All` is required to read/search the objects before adding them to the RMAU:

```powershell
$AdminUnitUserMembers = ("admThomas1@cloud-architekt.net","admScotty1@cloud-architekt.net")
$AdminUnitGroupMembers = ("prg_Lab-Tier1.Azure.1.PlatformOps", "prg_Lab-Tier1.Azure.1.SecOps")
$AdminUnitName = "Tier1-ManagementPlane.Azure"
$AdminUnitId = (Get-MgDirectoryAdministrativeUnit -Filter "displayname eq '$AdminUnitName'").Id

foreach ($AdminUnitUserMember in $AdminUnitUserMembers) {
    $UserId = (Get-MgUser -Filter "userPrincipalName eq '$($AdminUnitUserMember)'").Id
    New-MgDirectoryAdministrativeUnitMemberByRef -AdministrativeUnitId $AdminUnitId -AdditionalProperties @{"@odata.id"="https://graph.microsoft.com/v1.0/users/$UserId"}
}

foreach ($AdminUnitGroupMember in $AdminUnitGroupMembers) {
		$GroupId = (Get-MgGroup -Filter "displayName eq '$($AdminUnitGroupMember)'").Id
		New-MgDirectoryAdministrativeUnitMemberByRef -AdministrativeUnitId $AdminUnitId -AdditionalProperties @{"@odata.id"="https://graph.microsoft.com/v1.0/groups/$GroupId"}
}
```

The previous sample can be also customized to add different object types (user and groups) by `ObjectId` :

```powershell
$AzurePrivilegedObjects = ("182472f1-e301-4585-9cd8-78486ac9706a","1f4899f3-b288-4993-b664-6913a462a4db")
$AzurePrivilegedObjects | foreach-object { 
                $AdminUnitMember = (Get-MgDirectoryObject -DirectoryObjectId $_.ObjectId)
                Write-Host "Adding $($AdminUnitMember.AdditionalProperties.userPrincipalName) to $($AdminUnitName)"
                New-MgDirectoryAdministrativeUnitMemberByRef -AdministrativeUnitId $AdminUnitId -AdditionalProperties @{"@odata.id"="https://graph.microsoft.com/v1.0/directoryObjects/$($_.ObjectId)"}
}
```

*Side notes from my tests and automation jobs:*

- [Memberships of Administrative Units](https://registry.terraform.io/providers/hashicorp/azuread/latest/docs/resources/administrative_unit_member) can be also automated by Azure AD Terraform Provider.
- In the past, there was no `Remove-MgDirectoryAdministrativeUnitMember*` cmdlet in "Microsoft.Graph" PowerShell module. Therefor,e I’ve have been chosen the ["Delete" method on the API call](https://docs.microsoft.com/en-us/graph/api/administrativeunit-delete-members?view=graph-rest-1.0), like I did by using `Invoke-MgGraphRequest`.

### Identify which objects are protected by RMAU

The Azure (AD) portal shows a yellow information bar if a users or groups blade is restricted because of an assignment to a RMAU:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-06-13-administrative-units-restricted-management/rmau6.png)

*Restricted management will be shown in the overview of the "object" blade and in the properties.*

Microsoft Graph API offers also an opportunity to check if object management is restricted:

```powershell
$UserId = read-host -Prompt "Your user object ID to validate restricted management"
(Invoke-MgGraphRequest -Method Get -Uri https://graph.microsoft.com/beta/users/$userid -OutputType PSObject).isManagementRestricted
```

### Add or remove scoped directory roles to RMAU

The assignment of directory roles with scope on a RMAU can be automated by using an existing "Microsoft.Graph" cmdlet. Permissions on Application API Permission scope "RoleManagement.ReadWrite.Directory" or "Privileged Role Administrator" directory role assignment are required. In the following example, "User Administrator" will be assigned to Identity Team (Control plane) to manage user accounts in RMAU:

```powershell
# Associated Role Id to Directory Role can be listed with the following cmdlet
$DirectoryRole = "Groups Administrator"
$ScopedDirectoryRole = (Get-MgDirectoryRole -filter "DisplayName eq '$($DirectoryRole)'").Id
$ScopedDirectoryRoleMemberGroup = "prg_Lab-Tier0.AADTenant.0.IdentityOps"
$ScopedDirectoryRoleMemberId = (Get-MgGroup -Filter "DisplayName eq '$($ScopedDirectoryRoleMemberGroup)'").Id

$Body = @{
    RoleId = $ScopedDirectoryRole
    RoleMemberInfo = @{
        Id = $ScopedDirectoryRoleMemberId 
    }
}
New-MgDirectoryAdministrativeUnitScopedRoleMember -AdministrativeUnitId $AdminUnitId -BodyParameter $Body
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-06-13-administrative-units-restricted-management/rmau7.png)

*Assignment of RMAU-scoped directory roles works similar to Administrative Units in general*

Side Note: Eligible directory role assignments on scope of RMAUs are also supported by Azure AD PIM. Check out the Microsoft docs article if you are interested in using PIM and consider to including the RMAU Object Id in the scope of `directoryScopeId`

### Audit logs in Azure AD

Activities on RMAUs will be shown in a dedicated "OperationName" (e.g., "Add member to restricted management administrative unit"). This allows to use a simple filter for audit events related to RMAU operations. 

```powershell
AuditLogs
| where OperationName contains "restricted management administrative unit"
| extend InitiatingUserOrApp = iff(isnotempty(InitiatedBy.user.userPrincipalName),tostring(InitiatedBy.user.userPrincipalName), tostring(InitiatedBy.app.displayName))
| extend InitiatingUserOrAppId = iff(isnotempty(InitiatedBy.user.id),tostring(InitiatedBy.user.id), tostring(InitiatedBy.app.servicePrincipalId))
| extend InitiatingIpAddress = iff(isnotempty(InitiatedBy.user.ipAddress), tostring(InitiatedBy.user.ipAddress), tostring(InitiatedBy.app.ipAddress))
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-06-13-administrative-units-restricted-management/rmau8.png)

*Result of KQL query on Audit Logs to check change of assignments on RMAU*

Strictly monitoring of removing privileged or sensitive objects from RMAU should be considered.

Create, update or deleted operations on RMAUs are also available from the Audit Log . This sensitive activity can be filtered by using the property value of "IsMemberManagementRestricted":

```powershell
AuditLogs
| mv-expand TargetResources
| mv-expand TargetResources.modifiedProperties
| mv-expand TargetResources.modifiedProperties | where TargetResources_modifiedProperties.displayName == "IsMemberManagementRestricted"
| extend ActorId = iff(isnotempty(InitiatedBy.user.id),tostring(InitiatedBy.user.id), tostring(InitiatedBy.app.servicePrincipalId))
| extend ActorName = iff(isnotempty(InitiatedBy.user.userPrincipalName),tostring(InitiatedBy.user.userPrincipalName), tostring(InitiatedBy.app.displayName))
```

## Samples from my "Enterprise Access Model" implementations

In the following section, I would like to give you an overview about some use cases in a tiered administration environment where RMAUs offers a great security benefit in restricted management. Unfortunately, security groups (using "PIM for Groups") and role-assignable groups (Azure AD PIM) are not supported or appropriated. Using "PIM for Groups" and RMAU would be a perfect match to combine protection with the capability of Just-in-Time (JIT) access.
Nevertheless, there are a couple of scenarios where users are assigned to groups with eligible roles to Azure RBAC.

The described examples are part of my solution to implement an "Enterprise Access Model" and result of continuous research (started three years ago), development and experiences to find a practical approach to implement a tier model in Microsoft Azure and Azure AD.

*Are you interested in learning more about my implementation of Enterprise Access Model?
I’m speaking at TEC 2023 in a deep dive session about entire solution and share more insights how you can protect privileged identities in Azure AD. More information and tickets:* [The Experts Conference (TEC) 2023](https://theexpertsconference.cventevents.com/event/ff6b61dc-386d-422c-a142-8bd9b3b75453/websitePage:55dc8fd7-e500-4550-9055-6b1fde364777)

### Security principals in Azure RBAC role assignments

Protecting privileged users and groups on management and workload plane in Microsoft Azure was one of the previously named use cases. This is even more important, especially for those sensitive role definitions which have "Owner" or "Contributor" permission on Landing Zones (Workloads).

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-06-13-administrative-units-restricted-management/rmau9.png)

*Overview of my Enterprise Access and Tiered Administration Model.
Users and groups with permissions on the different tiered levels are assigned to the related RMAUs or AUs.
For example, user accounts with sensitive workload plane permissions (Contributor of Resource Group) are assigned to RMAU "Workload Plane Accounts"."*
_Side Note: I would recommend to use role-assignable groups as primary "role group" for comprehensive horizontal scope (such as "PlatformOps" or "SecOps") on management plane which offers also the capability to use this groups for delegation to scoped Azure AD roles (e.g., managing Landing Zone security groups in a RMAU)._

An overview and sample of my defined Azure roles and classification in the "Tired Administration" design can be found [here](https://github.com/Cloud-Architekt/AzureRBAC/blob/main/EAS_EAM_AzureRBAC_TabularSummary.pdf).

The privileged principals in Azure RBAC need to be protected and assigned to an associated RMAUs. Therefore, I build this automation as part of my AADOps PoC project.

**Identification of Azure RBAC privileged objects for assignment**

I’ve used [my existing PowerShell script](https://github.com/Cloud-Architekt/AzureRBAC/blob/main/Scripts/Get-AADObjectsFromAzureRBAC.ps1) to get a list of all eligible and permanent roles in Azure PIM.
In addition, I build a classification on role scope and actions to define the access (tier) level of the privileges.

The diagram below shows you an overview about the automation which I have developed to classify and assign privileged objects to RMAU automatically.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-06-13-administrative-units-restricted-management/rmau10.png)

*Example: Built-in or custom roles are classified as "Control plane" (Tier0) if they include action on "virtualMachines" and are assigned to the resource group which contains domain controllers.
Users or groups with related Azure RBAC assignments will be classified as "Control plane" administrators and assigned to the associated RMAU automatically. The certain role assignments will be also stored in JSON export which shows the assigned privileged RBAC role.*

At the end, this solution offers an automated assignment of privileged users and groups to the related "tier level" RMAUs. This provides an automation to restrict management for every classified privileged object as soon as the assignment has been made and the pipeline has triggered.

Azure Tags can be also used for classification of "AdminTierLevel" and "Service" alongside of manual classification in a JSON file.

*Alternate solution: Microsoft has been released an option to assign users to Administrative Units automatically by using a "[Dynamic membership rules](https://docs.microsoft.com/en-us/azure/active-directory/enterprise-users/groups-dynamic-membership)". This can be also used to assign Azure admins to RMAU if you have introduced a unique attribute to classify or identify those accounts (e.g., ExtensionAttribute or Naming Convention). Don’t forget to consider the trust of the  attributes used in the membership rules. Everyone who can modify this attribute will be able to manage assignments to the RMAU. Groups are not supported for [dynamic membership](https://docs.microsoft.com/en-us/azure/active-directory/roles/admin-units-members-dynamic) (rules) yet.*

*Note: [Custom security attributes](https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/custom-security-attributes-overview) cannot be used as for dynamic assignment.*

### Project (collection) members in Azure DevOps

Abuse of privileged service connections (defined in service connection) or manipulation of CI/CD pipelines can be achieved by "account takeover" from privileged "Project (Collection) Administrators" or members of sensitive projects in Azure DevOps.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-06-13-administrative-units-restricted-management/rmau11.png)

*Overview of sensitive users and groups in Azure DevOps organizations. Project collection administrators and other sensitive role members with direct/indirect privileges to manipulate protected branches or pipelines should be protected by RMAU.*

Modification of the assigned security groups from Azure AD is another access path.
Review sensitive permission assignments of your Azure AD objects on organization- and project-level. [Vinicius Moura](https://vinijmoura.medium.com/?source=post_page-----54f73a20a4c7-----------------------------------) has written a blog post about using Azure DevOps CLI to get [a list of all users and group permissions](https://vinijmoura.medium.com/how-to-list-all-users-and-group-permissions-on-azure-devops-using-azure-devops-cli-54f73a20a4c7) in Azure DevOps.

Members with privileged access on "Organization- and Collection-Level" should be also restricted (e.g., Project Collection Administrators). But also, users with assigned "Project- or Object-level" permissions should be considered. e.g., when you are using Azure DevOps Projects to automate Azure and Azure AD environment (e.g., M365DSC) and users can modify Branch Policies, Service Connection or any other sensitive action.

### Billing role members from Azure Enterprise Agreement Management

Users with roles in Enterprise Agreement (EA) portal are another use case for sensitive accounts without protection in Azure AD by default. Read my [Twitter threat](https://twitter.com/Thomas_Live/status/1511236926132604928) or [blog post](https://www.cloud-architekt.net/azure-ea-management-security-considerations/) if you want to learn more about these high-privileged roles.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-06-13-administrative-units-restricted-management/rmau12.png)

*Enterprise Administrators are able to modify roles in EA portal and change Account Owner which has full control on the Azure subscriptions. RMAU supports you to limit user management for those sensitive accounts as well.*

I’ve written a [PowerShell script](https://github.com/Cloud-Architekt/AzureRBAC/blob/main/Scripts/Get-AzEARoleMembers.ps1) to get a list of all assigned users in EA billing roles. This can be used (similar to the Azure RBAC script) to get a list of users which could be protected by assignment to RMAUs. The [Billing REST API](https://docs.microsoft.com/en-us/rest/api/billing/2019-10-01-preview/billing-accounts/list) and [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/billing/account?view=azure-cli-latest#az-billing-account-list) allows to get a list of users with access.

Review all users with EA roles and consider protecting those accounts by assigning them to the associated RMAU (e.g., Enterprise Administrator to "Tier0-ControlPlane.Azure").

### Administrative Accounts to manage B2C tenants

Organizations which have been implemented "Azure Active Directory B2C" tenants are mostly using accounts from the organization’s tenant for privileged access. Those accounts will be invited as B2B guest users and assigned to directory roles in the B2C tenant.

![abc.png]({{ site.url }}{{ site.baseurl }}/assets/images/2023-06-13-administrative-units-restricted-management/rmau16.png)

*User account in organization’s tenant will be invited to B2C tenants for management.*

Assignment of the directory role in the B2C tenant will not apply the protection of directory role members in the home tenant. Therefore, those accounts should be assigned to RMAU to avoid "account takeover" by "User Administrators" or other delegated Azure AD roles in the organization’s tenant. Identify those B2C admin accounts in your tenant and assign them to an RAMU (e.g., Tier0-ControlPlane.AzureAD-B2C). This prevents privilege escalation to customer identities from directory role members of the organization tenant.

### Azure AD configuration items

Security groups will be used in many cases for configuration or policy assignments.
For example, include/exclude groups from Conditional Access Policies but also for other policies such as Authentication Methods (scope for SSPR or TAP).

Identify sensitive groups based on the pre-defined naming convention for this configuration group. Modification of this group objects should be restricted to "Identity administrators" (Tier0/Control Plane) only. Otherwise, Intune or Windows 365 administrators can modify assignments of configuration items in your Azure AD tenant.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-06-13-administrative-units-restricted-management/rmau13.png)

*Exclusion Groups for Conditional Access will be created and assigned to RMAU. This prevents other directory roles with "Group Management" permissions to add their accounts in one of those groups to bypass Conditional Access policies.*

### Objects of Privileged or Secure Admin Workstations (PAW/SAW)

Azure AD device objects will be used in [Conditional Access "Device filters" to restrict access from PAW/SAW only](https://docs.microsoft.com/en-us/azure/active-directory/conditional-access/concept-condition-filters-for-devices#common-scenarios). In the past, it was a challenge to restrict Intune Administrators to manipulate device object attributes.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-06-13-administrative-units-restricted-management/rmau14.png)

*Device objects of PAW/SAW can be assigned to RMAU for restricted management by Intune and Windows 365 Administrators. [Azure AD custom roles](https://docs.microsoft.com/en-us/azure/active-directory/roles/custom-create) can be created and assigned to allow Endpoint admins limited management of the admin workstation.*

Security groups for Intune RBAC, Device Compliance or Configuration Profiles assignment of SAW/PAW devices are other sensitive objects which should be restricted.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-06-13-administrative-units-restricted-management/rmau15.png)

*This dynamic group will be used to assign settings to SAW devices for "Control plane" administrators.*

Microsoft Graph API can be used to [get a list of all users and groups with assigned Intune RBAC roles](https://docs.microsoft.com/en-us/graph/api/resources/intune-rbac-conceptual?view=graph-rest-1.0).