---
title: "Microsoft Entra Workload ID - Introduction and Delegated Permissions"
excerpt: "Workload identities will be used by applications, services or cloud resources for authentication and accessing other services and resources. Especially, organizations which follows a DevOps approach and high automation principals needs to manage those identities at scale and implement policies. In the first part of a blog post series, I would like to give an overview about some aspects and features which are important in delegating management of Workload ID in Microsoft Entra: Who can see and create apps? Why you should avoid assigning 'owner' to service principals or application objects?"
header:
  overlay_image: /assets/images/2023-08-08-entra-workload-id-introduction-and-delegation/workloadid-intro.png
  overlay_filter: rgba(102, 102, 153, 0.85)
  teaser: /assets/images/2023-08-08-entra-workload-id-introduction-and-delegation/workloadid-intro.png
search: true
toc: true
toc_sticky: true
categories:
  - Azure AD
  - Microsoft Entra
tags:
  - AzureAD
  - Microsoft Entra
  - Workload ID
  - Azure
last_modified_at: 2023-07-08
---

This blog post is part of a series about Microsoft Entra Workload ID:
- Introduction and Delegated Permissions
- Lifecycle Management and Operational Monitoring
- Advanced Monitoring and Security

## Introduction of Workload ID

Microsoft has introduced “Workload Identities” as overriding notion for non-human identities but also as “new product” as part of Microsoft Entra. The product name has been renamed from "Azure AD Workload Identities" to "Microsoft Entra Workload ID" in early July 2023. The premium licenses covers some features which has been available as public preview before (such as Conditional Access support for Service Principals). I will shown some use cases for the capabilities of Microsoft Entra Workload ID in the next part of this blog series.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-08-entra-workload-id-introduction-and-delegation/workloadid-intro-deleg0.png)

*Workload Identity Premium features in the Microsoft Entra Portal.
You’ll find a overview about the features and capabilities which are included in Microsoft Entra Workload ID Premium and which ones are free at the [Microsoft Entra Workload ID FAQ page](https://learn.microsoft.com/en-us/azure/active-directory/workload-identities/workload-identities-faqs#what-features-are-included-in-workload-identities-premium-plan-and-which-features-are-frees).*

Provisioning and general usage of Service Principals are still free in Microsoft Entra ID (formely known as Entra ID), only certain new capabilities are part of the new “premium plan” for Workload ID.

*An overview about the definition of workload identities is well documented in the Microsoft Learn article “[What are workload identities?](https://learn.microsoft.com/en-us/azure/active-directory/develop/workload-identities-overview#supported-scenarios)”.*

### Common deployment and integration scenarios

The following common use cases should provide some examples of the terminology and different types of workload identities in Microsoft Entra ID:

- **Application identities** will be created in the “App Registration” blade and used for integration of applications for using modern authentication and/or access to other Entra ID-protected resources. They can be also used for multi-tenancy, which allows to provide access to an application outside of the own organization (common for SaaS scenario). Access to the resources can be assigned with delegation or application API permissions and requires a [consent from the admin or user](https://learn.microsoft.com/en-us/azure/active-directory/manage-apps/user-admin-consent-overview).
    - **[Delegated API Permissions](https://learn.microsoft.com/en-us/graph/auth-v2-user)** allows the application to get access on behalf of the signed-in user. This will be mostly used for interactive sessions and limited to the scope of the specific user.
        
        ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-08-entra-workload-id-introduction-and-delegation/workloadid-intro-deleg1.png)
        
        Example: Application accessing Microsoft Graph API with scoped and delegated token of the user
        
    - **[Application API Permissions](https://learn.microsoft.com/en-us/graph/auth-v2-service)** enables the application to access a resource without signed-in user and will be typically used for background or automation workflows.
        
        ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-08-entra-workload-id-introduction-and-delegation/workloadid-intro-deleg2.png)
        
        Example: Application accessing Microsoft Graph API with permissions assigned to the Application object.
        
    
    Authorization to other APIs or roles can be granted with assigning the service principal to the target RBAC system (e.g., Azure RBAC for managing resources, Entra ID admin roles for scoped permissions on directory objects).
    
- **[Managed Identities](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)** is a recommended way for integration of workloads running in Azure or Azure Arc-connected environments (including GCP, AWS and on-premises infrastructure). This identity can be assigned to a particular Azure-managed resource and will be bounded to the lifecycle of this object (”**[System-assigned](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)**”). A standalone identity can be also created which allows to use them by multiple resource (”**[User-assigned](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/how-manage-user-assigned-managed-identities?pivots=identity-mi-methods-azp)**”). Furthermore, you can also use a “**[Workload Identity Federation](https://learn.microsoft.com/en-us/azure/active-directory/workload-identities/workload-identity-federation)**” to integrate workloads from other platforms by establishing a trust to an external Open ID Connect issuer / Identity Provider (IdP) (such as GitHub Actions, Google Cloud). In general, there’s no need to manage credentials for those workloads because of taking benefit of Azure as Management Plane to auto-manage credentials or establishing federation to a trusted IdP. This feature is also available for “Application Identities” and eliminates the requirement for managing credentials as well.
    
    Application API Permissions and also authorization as part of RBAC assignments can be granted to Managed Identities. As far as I know, calls on using delegated permissions are not applicable. The management of API permissions is restricted to Graph API (no support in Portal UI).
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-08-entra-workload-id-introduction-and-delegation/workloadid-intro-deleg3.png)
    
    Example: Managed Identity has been assigned or federated to a workload. Assignment to an Azure RBAC role exists allows to call the Azure Resource Manager API for deployment of resources.
    
    *Side Note: Managed Identities are out of scope for this article but I’m already planning a dedicated article about the advantages and considerations in using Managed Identities.*
    

The following summary table gives you a quick overview about the different types of workload identities

| Type | Application Identity (with Key or Certificate) | Application Identity (with Federated Credential) | Managed Identity (System-Assigned) | Managed Identity (User-Assigned) |
| --- | --- | --- | --- | --- |
| Use Cases | No limitations | Limited, support workload and/or IdP required | Limited, support, workload must be a Azure-Managed resource | Limited, support, workload must be a Azure-Managed resource or federated credential |
| Security Boundary | Single- or Multi-Tenant | Single- or Multi-Tenant | Single Tenant, limited Multi-Tenant access* | Single-Tenant, limited Multi-Tenant access* |
| Relation to workload | No relation or allocation to workload or resource | Relation to Issuer/Entity of the federated provider | Allocation to resource 1:1 | Allocation to resource N:1 |
| Lifecycle Management | Managed by admin or automated process | Managed by admin or automated process | Managed by Azure  | Standalone Azure resource (managed by admin or automated process) |
| Secret Management | Key or Certificate Renewal required | No particular secret rotation required | Managed by Azure | Managed by Azure |
| Token Lifetime / Cache | 1h (Default), 24h (CAE) | less than or equal to 1h | [up to 24h](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/managed-identity-best-practice-recommendations#limitation-of-using-managed-identities-for-authorization) | [up to 24h](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/managed-identity-best-practice-recommendations#limitation-of-using-managed-identities-for-authorization) |

*Single Tenant Application only, but access can be granted to Azure RBAC and resources in other tenants via Azure Lighthouse

### Challenges in managing Non-Human Identities

Microsoft has shared some interesting statistics at the “[2023 State of Cloud Permissions Risks Report](https://query.prod.cms.rt.microsoft.com/cms/api/am/binary/RW10qzO)”. This includes also the ratio between user and workload identities but also the numbers of stale identities:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-08-entra-workload-id-introduction-and-delegation/workloadid-intro-deleg4.png)

*Summary of related statics about workload identities. Source: Microsoft “2023 State of Cloud Permissions Risks Report”*

It shows an incredible number of non-human identities which will certainly used for automation with sensitive access to IT assets (e.g., provisioning of Azure resources, Microsoft Sentinel SOAR, automation for IAM workflows).

But on the other side, most organization are still facing challenges in managing a Workload ID compared to user accounts. The following risks are (in my opinion) widely present:

- **No defined lifecycle management**
HR data and processes can be used as source of authority for user lifecycle. Many organization are not able to build a similar relation between IT asset/cloud resource and workload identity.
- **Risks of leaked secrets or compromised credentials**
Multi-factor authentication and strong (or Passwordless) authentication can be implemented for privileged users but can not be adopted for Workload ID. Most organization are using secrets for Service Principals authentication because other credential types are not supported or having limited knowledge about alternate options. Credentials will be handed out to developers for application or DevOps integration.
- **Escalation paths by standing and overprivileged access**
Review of used permissions or credentials are rarely seen and will not be conducted frequently. Options to assign time-bound access are very limited. In addition, missing concepts for (scoped) delegation are leading to an increased risk of privilege escalation paths, uncontrolled and rampant usage without governance rules.
- **Lack of auditing, monitoring and detections in Security Operations**
Sign-in reports of service principals has been introduced in September 2020. Nevertheless, it seems that those identities will not be monitored on the same quality level as human identities.

### Entra ID Objects of Application Identities

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-08-entra-workload-id-introduction-and-delegation/workloadid-intro-deleg5.png)

**Application object** has been created in the “App Registration” blade and defines the API permissions, credentials and general properties (incl. authentication settings).
An instance of the application has been created based on the Application object and will be represented by the **Service Principal.** It exists in the local directory (”home tenant”) where the application was registered but also in other tenants (”resource tenant”) in case of multi-tenant application. This object will be used for assigning memberships to Entra ID groups but also role assignments in other RBAC systems.

A service principal can be modified in the resource tenant and issued with different credential than the application objects in the home tenant.

Owners can be assigned to the Application and Service Principal objects which delegates to user full control for managing the properties and credentials.

**Application (Client) ID** represents the “login name” of a workload identity and is shared between object types. A shared secret, a signed JWT token (from the private key of a certificate) or a token (from a federated identity) is required to request a token from the Entra ID endpoint. Implementation of Microsoft Authentication Libraries allows easy code integration for handling token request, validation (of signature) and renewal.

### Entra ID Objects of Managed Identities

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-08-entra-workload-id-introduction-and-delegation/workloadid-intro-deleg6.png)

Managed Identities exists as **Resource in Azure** but also as Service Principal in the associated Entra ID Tenant. There’s no (visible) Application object for Managed Identities. **Certificates as credentials are created and maintained by Microsoft** on the Service Principal-Object. Assignments to API Permissions needs to be configured on this object type as well. The name of the Enterprise App (Service Principal) is equal to the related Azure Resource Name.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-08-entra-workload-id-introduction-and-delegation/workloadid-intro-deleg7.png)

Certificates of Managed Identities have an expiration of 90 days and will be rolled after 45 days by Azure. More details on limitations and technical backgrounds are [described in the FAQ section on Microsoft Learn](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/managed-identities-faq).*

## Default User Permissions and Risks of Unmanaged Workload ID

But just who can read and create the related workload identity objects in a tenant? Let’s have a closer look to the default permissions for Entra ID users.

### Enumeration and Visibility of Properties

By design, all members have the default permission to read properties of the Application and Service Principal objects, including their granted permissions. But the enumeration of all objects is available for members only. External users will get also the permission to list all application when one of the following conditions met:

- No [guest user access restriction](https://learn.microsoft.com/en-us/azure/active-directory/enterprise-users/users-restrict-guest-permissions) by choosing “Guest users have the same access as members (most inclusive)” in the External collaboration settings
- External user has been changed from user type “Guest” to “Member”
- Assignment of built-in (e.g., “Directory Readers”) or custom Entra ID role (with permission to read properties of the objects) [to allow browsing all objects](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-external-users#guest-user-cannot-browse-users-groups-or-service-principals-to-assign-roles) in the tenant.
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-08-entra-workload-id-introduction-and-delegation/workloadid-intro-deleg8.png)
    
    Example of “[Custom Entra ID role](https://learn.microsoft.com/en-us/azure/active-directory/roles/custom-create)” which allows every assigned member to read properties of related Application Identities object. Role members are able to access Azure or Entra portal when [user access is restricted](https://learn.microsoft.com/en-us/azure/active-directory/fundamentals/users-default-permissions#restrict-member-users-default-permissions).*
    

### Create and Ownership of Applications

**Default User Role Permission**

By default, members are able to register applications and create an Application object in the tenant. In addition, the creator becomes owner of the application and enterprise application when using the Azure (AD) or Microsoft Entra Portal.

However, the users’ default permissions can be restricted and should be disabled in my opinion. Otherwise, you will allow the provisioning and management of workload identities by any member account.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-08-entra-workload-id-introduction-and-delegation/workloadid-intro-deleg9.png)

_Disable default permission to register application should be disabled and particular delegated to developer accounts or integrated to a managed workload identity lifecycle._

This setting is named `allowedToCreateApps` and will be defined in the `authorizationPolicy` which can be also managed in Microsoft Graph API:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-08-entra-workload-id-introduction-and-delegation/workloadid-intro-deleg10.png)

****Authorization Policy allows to restrict a few default member permissions.****

As you can see, another entry of the `authorizationPolicy` been marked in the screenshot.
An [app consent policy](https://learn.microsoft.com/en-us/azure/active-directory/manage-apps/manage-app-consent-policies?pivots=ms-powershell) will be defined for members to govern the permissions for user consent.
The value `permissionGrantPoliciesAssigned` is shown the “Policy Id” of a built-in (begins with “microsoft-”) or a custom policy. The default setting will be also shown in the [Portal UI](https://portal.azure.com/#view/Microsoft_AAD_IAM/ConsentPoliciesMenuBlade/~/UserSettings):

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-08-entra-workload-id-introduction-and-delegation/workloadid-intro-deleg11.png)

**Consent and permissions can be configured in the Azure portal but without advanced options (such as assigning a custom policy).**

_Side Note: Configuration of User and Admin Consent Permissions (incl. Policy Framework) is an essential and huge topic for identity security. If you are interested in to learn more about the attack scenarios and security configuration, check out the Azure AD Attack & Defense playbook about “[Consent Grant](https://github.com/Cloud-Architekt/AzureAD-Attack-Defense/blob/main/ConsentGrant.md)”._

## Entra ID admins roles and ownership for delegation

### **Application Developer**

Permission to register application can be delegated by assigning “[Application Developer](https://learn.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#application-developer)” role.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-08-entra-workload-id-introduction-and-delegation/workloadid-intro-deleg12.png)

***Built-in role “Application Developer” allows members to create application identities even the default user permission has been restricted***

Every created application by this role members will assign “Owner” permissions (by default) to the creator. In this way, the creator has full management permission on this object with a permanent and user-based assignment. External users can be also assigned as “Owner” but needs to have “Directory Reader” or “Member” type permission to take advantage of the permissions.

### **Risks of owner on application and service principal objects**

There are a couple of reasons why ownership has some disadvantages and new/existing assignment should be avoided:

- Owners can not be assigned to security groups and can not covered by Identity Governance for access review or Entitlement Management. This leads to permanent privileges to a single user account with limited visibility and management.
    - Limited visibility during maintenance as part of joiner/leaver/mover process
- No particular options to cover them in Conditional Access Policies (with Authentication Context or scope on Directory Role Assignment)
- Owner can not be granted as Eligible Assignment with Approval with Entra ID PIM
- Limited visibility as privileged role assignment (owners are not automatically protected as privileged user in Entra ID, compared to Entra ID role members)

Ownership includes permission to issue credentials and modify Redirect URI which is in particular interest of attackers to gain persistence access (and create “backdoors”) or stealing tokens with delegated permissions (by manipulation of replay URL). Microsoft has also given some note in the article about “[Overview of enterprise application ownership in Microsoft Entra ID](https://learn.microsoft.com/en-us/azure/active-directory/manage-apps/overview-assign-app-owners)”.

> The application may have more permissions than the owner, and thus would be an elevation of privilege over what the owner has access to as a user. An application owner can create or update users or other objects while impersonating the application. The elevation of privilege to owners can raise a security concern in some cases depending on the application's permissions.
> 

### (Cloud) Application Administrators

Built-in roles “[Application Administrator](https://learn.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#application-administrator)” and “[Cloud Application Administrator](https://learn.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#cloud-application-administrator)” allows to manage application and service principal object in the tenant. This affects all Application and Service Principal objects, except App Proxy for the particular role of “Cloud Application Administrator”.

This allows assigned members to takeover any provisioned non-human identities in your tenant and their assigned permissions. Therefore, this is a high-sensitive role assignment and should be considered as “Tier0” (Control Plane) administrators.

In addition, members of this Entra ID admin role are assigned to the consent policy “ `microsoft-application-admin`" which allows to grant tenant-wide admin consent on specific conditions.

But there’s also some other built-in roles which allows manage Application Identity objects, this includes the following roles:

- Partner Tier1/Tier2 support
- Hybrid Identity Administrator
- Directory Synchronization Accounts

Side Note: The previous named directory roles are mostly out of scope by strong Conditional Access Policies. For example: Hybrid Identity Admin will be used on AAD Connector and excluded from Device Compliance. Therefore you should monitor activities from those role members particular.

Monitor all Entra ID admin roles and assignments which includes (for example) the following permissions:

- microsoft.directory/applications/credentials/update
- microsoft.directory/applications/owners/update
- …

### Assigning Entra ID admin roles to scope of object-level

As already described, previous roles have sensitive permissions which should avoided to be granted on directory-level. It’s possible to assign the “Cloud Application Administrator” role on object-level of the application and service principal:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-08-entra-workload-id-introduction-and-delegation/workloadid-intro-deleg13.png)

***Role assignment of “Cloud Application Administrator” allows to select scope on object-level***

This gives you also the opportunity to replace owner permission with object-level permissions to the Entra ID role. This includes all benefits of using Entra ID roles, such as eligible membership (managed by PIM) or group-based assignment. 

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-08-entra-workload-id-introduction-and-delegation/workloadid-intro-deleg14.png)

**Eligible Assignment of “Cloud Application Administrator” on scope of app registration “b2xapp”.**

### Custom Entra ID admin roles for delegation

Microsoft has integrated many permissions related to Application and Service Principals objects to Entra ID Custom Roles. This allows us to create a custom role on specific permissions and follow the least-privilege approach. In addition, the custom role can be also assigned to all directory- and object-level.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-08-entra-workload-id-introduction-and-delegation/workloadid-intro-deleg15.png)

***Delegation of specific tasks can be achieved by custom roles (e.g., creating role permission set to managed application properties without updating credentials).***

### Permission to create and modify Managed Identities

The previous descriptions are mostly in relation to Application Identities in Entra ID. But who can create an managed identity in Azure?

Every “Contributor” or resource-specific Contributor role (such as Virtual Machine Contributor) are allowed to enable system-assigned identity for the related resources. There are no specific Entra ID permissions needed to **provision a system-enabled managed identity**.
Example: Enable managed identity for a Virtual Machine needs only a role action as `Microsoft.Compute/virtualMachines/write` which is included in the role definition of “Virtual Machine Contributor”.

There are two **specific roles for managing user-assigned identities**:

- [Managed Identity Contributor](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#managed-identity-contributor)
Create, Read, Update, and Delete User Assigned Identity
- [Managed Identity Operator](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#managed-identity-operator)
Read and Assign User Assigned Identity

Both roles have the permission to assign a managed identity to another resource. The related Resource Provider and Action namespace is named `Microsoft.ManagedIdentity` and can be also used for creating custom Azure RBAC roles. Other roles with [a wildcard on the action scope](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-definitions#role-definition-example) (such as Owner and Contributor) has also the permissions to modify and assign user-assigned identities.

***Next: Implementing a Lifecycle and Operational Monitoring***

*In the second part of the blog post series, I will describe some aspects which could be included in your strategy to implement a lifecycle monitoring. There are also solutions by Microsoft and the community which helps you to monitor your workload identities. The focus will be set on application identities because of the missing relation to a workload and credential management.*