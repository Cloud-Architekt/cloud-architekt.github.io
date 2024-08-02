---
title: "Identify and prevent abuse of  Managed Identities with Federated Credentials from unauthorized entities"
excerpt: "In this article, I would like to point out options to identify, monitor and avoid persistent access on Managed Identities privileges by adding federated credentials on User-Assigned Managed Identities (UAMI) from malicious or unauthorized entities. We will also have a quick look at attack paths and privileges which should be considered."
header:
  overlay_image: /assets/images/2024-08-02-identify-prevent-abuse-uami-federatedcredentials/UamiFedCreds1.png
  overlay_filter: rgba(102, 102, 153, 0.85)
  teaser: /assets/images/2024-08-02-identify-prevent-abuse-uami-federatedcredentials/UamiFedCreds1.png
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
  - Microsoft Sentinel
last_modified_at: 2024-08-02
---

## What are Federated Credentials?

A few years ago, Microsoft introduced federated credentials as an alternate option to client secrets or certificates for workloads outside of Azure. This requires establishing a trust relationship between a workload identity in Microsoft Entra to identify a token (by claims) from an external IdP. In the end, the workload outside of Azure will be able to use an access token in scope of the trust relationship to gain an access token from Microsoft Entra ID. How it works in details is very well explained in [Microsoft Learn](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation). You’ll find different ways to configure a managed identity with federated credentials in [Microsoft Learn](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation-create-trust-user-assigned-managed-identity?pivots=identity-wif-mi-methods-azp). 

Originally, this feature was limited to App Registrations in Microsoft Entra ID. Around one year ago, support for federated credentials on user-assigned identities (UAMI) has been introduced. In comparison to App Registrations, the federated credentials will be added on the Service Principal (Enterprise App) object and the default token lifetime is 24h (by default). During my tests, I was not able to find a successful way to abuse privileges in Microsoft Entra to modify credentials on the service principal object of the UAMI by Microsoft Graph API call. Managing federated credentials on UAMI requires privileges in Azure RBAC and will be executed by Azure Resource Manager (ARM) APIs. Federated credentials are not available for System-assigned Managed Identities.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-08-02-identify-prevent-abuse-uami-federatedcredentials/UamiFedCreds0.png)

*Overview of issued Federated Credentials on User-Assigned Managed Identities (UAMI)*

| | App Registration with Federated Credentials | User Assigned Managed Identity with Federated Credentials |
| --- | --- | --- |
| Delegation and Ownership | Application/Enterprise App Owner, Entra ID Role (Directory, Object) | Azure RBAC Role/Resource Owner |
| Visibility by default permissions | [Read by members](https://learn.microsoft.com/en-us/entra/fundamentals/users-default-permissions#compare-member-and-guest-default-permissions), [Limited read by guests](https://learn.microsoft.com/en-us/entra/fundamentals/users-default-permissions#compare-member-and-guest-default-permissions) ([depending on Restrictions](https://learn.microsoft.com/en-us/entra/fundamentals/users-default-permissions#restrict-guest-users-default-permissions)) | Requires read permission on resource scope |
| Recovery Options | [Soft deleted](https://learn.microsoft.com/en-us/entra/identity-platform/howto-restore-app) | [Soft deleted](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/delete-recover-faq#are-managed-identities-soft-deleted-) |
| Logging of operational changes | Entra ID Audit Logs | Azure Activity logs |
| Logging of sign-in activity | Yes | Yes (limited set of sign-in properties, such as IP Address) |
| Policies to restrict subject/issuer for issuing credentials | No, App Management policies seems to cover only [passwordCredentials and keyCredentialConfiguration](https://learn.microsoft.com/en-us/graph/api/resources/appmanagementconfiguration?view=graph-rest-1.0) | Azure Policies |
| Support for Entra ID Protection risk detections² | Yes (for single-tenant) | [Not supported](https://learn.microsoft.com/en-us/entra/id-protection/concept-workload-identity-risk) |
| Support for Conditional Access² | Yes (for single-tenant) | [Not supported](https://learn.microsoft.com/en-us/entra/identity/conditional-access/workload-identity) |
| Token Lifetime / Cache | 1h (Default), 24h (CAE) | up to 24h |

_Comparison of using Federated Credentials by App Registrations or UAMI_

² requires Workload ID Premium


# Required privileges and attack scenarios

As already described, privileges in Azure RBAC are required to add, update and remove the federated credentials on a UAMI. The resource actions `Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials/write` must be included in a built-in or custom role to be authorized for the associated operations.

The following KQL query can be used in Azure Resource Graph (ARG) to get a list of role definitions which includes an explicit privilege on resource `Microsoft.ManagedIdentity/userAssignedIdentities`

```jsx
AuthorizationResources
| where type =~ "microsoft.authorization/roledefinitions"
| mv-expand parse_json(properties.permissions)
| mv-expand parse_json(properties_permissions.actions)
| where properties_permissions_actions contains "Microsoft.ManagedIdentity/userAssignedIdentities"
| project ResourceId = id, RoleName = properties.roleName, RoleType = properties.type, AssignableScope = tostring(properties.assignableScopes), Description = properties.description, isServiceRole = properties.isServiceRole, RoleAction = properties_permissions_actions
```

In addition, the following built-in roles have also full control on `userAssignedIdentities` because of comprehensive (wild card) authorization in the role definition:

- Contributor (incl. Lighthouse Delegations)
- Owner (incl. [AOBO](https://learn.microsoft.com/en-us/partner-center/billing/partner-earned-credit-troubleshoot#how-to-verify-aobo-permissions) permissions by CSP)
- Co-Administrator and Service Administrators (retired on August 31, 2024)

In my recent community talks about Azure Governance and Workload Identities, I’ve also described some attack scenarios on Federated Credentials. In general, there are two main options for attackers to abuse this authentication method of workload identity:

**1) Compromising workload and/or IdP of the trusted entity** which is defined in the federated credentials. Everyone who owns the workload or 3rd Party IdP will have the opportunity to gain an access token from Entra ID in exchange of the federated credential. It is in the nature of this technical design/approach and should be obvious that we need to secure the workload and establish a trust relationship to trustworthy IdPs. Token replay was one of the examples in my community talks to underline the importance of measures for securing workload environment (e.g., GitHub runners) but also direct/indirect access to dependencies of executing workload code. 

**2) Compromising any security principal with access** to a managed identity. This can also include a CSP provider which has access to a subscription by Azure Lighthouse. Initial attack by phishing on developers with privileged access or compromised CI/CD are just a few other scenarios that seems certainly possible. In the past, this attack scenario has already existed and allows to gain access tokens from a Managed Identity. However, it was required to also gain workload access to the associated resource (e.g., Virtual Machine or Azure Function). A trust relationship was established between resources which exist within your Azure and your tenant boundary. By adding federated credential to any UAMI, it seems to be easier to add a kind of “backdoor” because of underrated impact of privileges and uncovered auditing and/or monitoring of this sensitive resources. Other sign-in activities or operations to the trusted entity of the federated credential could also be outside of your control and monitoring. Most of the trusted (external) entities (e.g., GitHub or any other 3rd Party IdP) are managed outside of the tenant boundary and may not be covered or visible by security operations.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-08-02-identify-prevent-abuse-uami-federatedcredentials/UamiFedCreds%201.png)

*Overview of attack paths to gain access to UAMI for adding or modify federated credentials*

Dirk-jan Mollema has written an [excellent blog post](https://dirkjanm.io/persisting-with-federated-credentials-entra-apps-managed-identities/) about detailed steps to create an OpenID connect provider ([roadoidc](https://github.com/dirkjanm/ROADtools/tree/master/roadoidc)) by using the tool ROADtools. He also explains the risks and steps how an attacker could abuse federated credentials which also applies to UAMI. I can only recommend to reading his awesome article:

- [Persisting on Entra ID applications and User Managed Identities with Federated Credentials - dirkjanm.io](https://dirkjanm.io/persisting-with-federated-credentials-entra-apps-managed-identities/)

# Analysis of existing UAMI with federated credentials

## Creating inventory and detailed report

There are a couple of ways to identify managed identities by building custom reports or queries.
However, various request to Azure Resource Manager (ARM) or Azure Resource Graph API but also Microsoft Graph API are needed to get a full visibility of a user-assigned managed identity and their relation to federated credentials but also assigned privileges.

In my opinion, the most comprehensive solution offers the free community tool [AzADServicePrincipalInsights](https://github.com/JulianHayward/AzADServicePrincipalInsights) by [Julian Hayward](https://www.linkedin.com/in/julianhayward/). The report can be exported as HTML, JSON and CSV. It includes also important details which helps to identify if the federated credentials have been assigned to critical privileges:

- Azure role assignments
- Application API permissions (e.g., Microsoft Graph)
- Exposed API roles
- Entra ID role assignment
- Ownership to objects

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-08-02-identify-prevent-abuse-uami-federatedcredentials/UamiFedCreds%202.png)

*Detailed reporting of Federated Credentials by using AzADServicePrincipalInsights*

## Ingest report data to Microsoft Sentinel/Log Analytics for enrichment and hunting

I have implemented together with Julian also the capability to ingest the data to a custom table in Microsoft Sentinel. The required steps are described in my previous blog post about [Advanced Detections and Enrichment in Microsoft Sentinel](https://www.cloud-architekt.net/entra-workload-id-advanced-detection-enrichment/#integration-of-azadserviceprincipalinsights-as-custom-table) for Workload IDs.

This allows to use the data for hunting but also enrichment of detections which will be described in the next section of this article.

**Example 1: Get all UAMIs with Federated Credentials including details of trusted issuer and subject**

```jsx
AzADServicePrincipalInsights_CL
| summarize arg_max(TimeGenerated, *) by ObjectId
| mv-expand parse_json(ManagedIdentityFederatedIdentityCredentials)
| where ManagedIdentityFederatedIdentityCredentials.type == "Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials"
| project ObjectId, SP[0].SPDisplayName, SPOwnedObjects, SPAADRoleAssignments, SPAppRoleAssignments, SPAppRoleAssignedTo, SPAzureRoleAssignments, ManagedIdentityFederatedIdentityCredentials
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-08-02-identify-prevent-abuse-uami-federatedcredentials/UamiFedCreds%203.png)

**Example 2: Get all UAMI Federated Credentials with access to Application Permissions, Azure or Entra ID Roles**

```jsx
AzADServicePrincipalInsights_CL
| summarize arg_max(TimeGenerated, *) by ObjectId
| mv-expand parse_json(ManagedIdentityFederatedIdentityCredentials)
| where isnotempty(SPAADRoleAssignments) or isnotempty(SPAppRoleAssignments) or isnotempty(SPAzureRoleAssignments)
| where ManagedIdentityFederatedIdentityCredentials.type == "Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials"
| project ObjectId, SP[0].SPDisplayName, SPOwnedObjects, SPAADRoleAssignments, SPAppRoleAssignments, SPAppRoleAssignedTo, SPAzureRoleAssignments, ManagedIdentityFederatedIdentityCredentials
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-08-02-identify-prevent-abuse-uami-federatedcredentials/UamiFedCreds%204.png)

**Example 3: Filtering critical Entra ID roles which are granted to UAMI with Federated Credentials**

```jsx
AzADServicePrincipalInsights_CL
| summarize arg_max(TimeGenerated, *) by ObjectId
| mv-expand parse_json(ManagedIdentityFederatedIdentityCredentials)
| where ManagedIdentityFederatedIdentityCredentials.type == "Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials"
| project ObjectId, SP[0].SPDisplayName, SPOwnedObjects, SPAADRoleAssignments, SPAppRoleAssignments, SPAppRoleAssignedTo, SPAzureRoleAssignments, ManagedIdentityFederatedIdentityCredentials
| mv-expand parse_json(SPAADRoleAssignments)
| where SPAADRoleAssignments.roleIsCritical == false
| project SPAADRoleAssignments.roleDefinitionName, SPAADRoleAssignments.resourceScope, SPAADRoleAssignments.roleDefinitionDescription, SP = SP_0_SPDisplayName, ManagedIdentityFederatedIdentityCredentials.name, ManagedIdentityFederatedIdentityCredentials.id
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-08-02-identify-prevent-abuse-uami-federatedcredentials/UamiFedCreds%205.png)

## Auditing by built-in Azure Policy

You can also audit issued federated credentials by using a built-in Azure Policy set (in preview) with the name “Managed Identity Federated Credentials should be of approved types from approved federation sources”. This allows you to integrate natively the results as part of the security recommendations for your Azure environment. It provides you with a simple view on federated credentials which have been configured outside of “allowlisted” and trusted federated provider.
I will explain how to use this policy for prevention later in this article.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-08-02-identify-prevent-abuse-uami-federatedcredentials/UamiFedCreds%206.png)

*Policy compliance with list of issued federated credentials and check on allowed issuer types*

# Detection on authorized actors

Operations to add federated credentials will be audited in the `AzureActivity` (Microsoft Sentinel/Diagnostic Logs), `CloudAppEvents` and `CloudAuditEvents` (both available in Microsoft Defender XDR). It seems there are no details included of the federated credentials (such as issuer, entity type or subject). Therefore, only related information about the actor which creates the federated credential can be used for alerting by default. In addition, enrichment or hunting could be used to identify if the trust relationship has been established to an unauthorized or malicious entity. This can be achieved by using a playbook to enrich data from Azure Resource Graph about the Managed Identity or include a query to (fresh updated) AzADSPI exported data.

The following analytics rule template is written for Microsoft Sentinel and offers to include an allow list of actors (by group membership) or roles of actors but also set the scope which should be included for monitoring. Furthermore, risk level by Microsoft Entra ID Protection will be included to set a scope (optional) on compromised identities.

- Unauthorized actor has been added Federated Credential on User-Assigned Managed Identity [[ARM Template]](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Detections/EID-WorkloadIdentities/Unauthorized%20actor%20has%20been%20added%20Federated%20Credential%20on%20User-Assigned%20Managed%20Identity.json) [[YAML format]](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Detections/EID-WorkloadIdentities/Unauthorized%20actor%20has%20been%20added%20Federated%20Credential%20on%20User-Assigned%20Managed%20Identity.yaml)

The parameters will be defined in the query but can also be outsourced to a WatchList for better management of the allow list. A similar query can also be written for Microsoft Defender XDR because of the availability of audit events in the table of `CloudAppEvents` and `CloudAuditEvents`.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-08-02-identify-prevent-abuse-uami-federatedcredentials/UamiFedCreds%207.png)

*The result of the analytics rule in Microsoft Sentinel can be triggered in the following incident*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-08-02-identify-prevent-abuse-uami-federatedcredentials/UamiFedCreds%208.png)

*Entity type “Azure Resource” shows the `federatedIdentityCredentials`  and allows to use a deep link to navigate to the resource page with essential properties.*

# Detection of suspicious sign-ins by federated credentials

Sign-in by using App Registrations with federated credentials will be audited in `AADServicePrincipalSignInLogs` and includes an ID for the federated credential ("FederatedCredentialId”). This allows us to get easily a list of all sign-ins by using federated credentials including the associated IP Address.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-08-02-identify-prevent-abuse-uami-federatedcredentials/UamiFedCreds%209.png)

I was not able to find any similar property in the `AADManagedIdentitySignInLogs`.
In general, the IP Address is also missing in the sign-in logs for Managed Identities.
So in this case, we have only limited capabilities to write detections or start hunting on suspicious sign-ins for using federated credentials on UAMI.

# Hunting of activities by compromised UAMI

As already mentioned, access tokens of UAMI have a lifetime up to 24h. Removing federated credentials has not revoked issued tokens (for example, as trigger of CAE). It seems that CAE does not support Managed Identities as documented in [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation-workload). The actor can still be used within the long-lived token even if a malicious federated credential has been  removed. However, you can use `UniqiueTokenIdentifier` from AADManagedIdentitySignInLogs to track the activities by valid tokens in other logs (e.g., AzureActivity).

```jsx
AADManagedIdentitySignInLogs
| where ServicePrincipalName == "<CompromisedUamiDisplayName"
| join kind=inner ( AzureActivity
    | extend UniqueTokenIdentifier = tostring(parse_json(Claims).uti)
) on UniqueTokenIdentifier
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-08-02-identify-prevent-abuse-uami-federatedcredentials/UamiFedCreds%2010.png)

*Hunting of  activities by issued (long-lived) token by compromised UAMI*

# Prevention by Azure Policies and Governance

There are a few actions and steps that should be considered to avoid unauthorized trust relationships or usage of federated credentials

- **Assign least privileges for Landing Zone Owners**
If possible, avoid assigning Owner, Contributor any role including write `Microsoft.ManagedIdentity/*` on sensitive workloads to Landing Zone Owners. There are many job-function roles which could fit to the least privileges and required permissions for DevOps. Microsoft Entra Permissions Management could also support you to create a custom role based on the required role actions. Monitor any custom role which includes sensitive role actions on UAMI.
- **Deny assignment of federated credentials where it is not in use**
Microsoft already described in the “[considerations](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation-considerations#azure-policy)” section of Workload Identity Federation but also in this [detailed instruction article](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation-block-using-azure-policy) how to create a deny Azure Policy. This simplified policy definition would allow to block the usage of federated credentials in the assigned scope.
- **Audit compliance and restrict federated credentials on defined and trusted sources**
A [built-in Azure policy set](https://www.azadvertizer.net/azpolicyinitiativesadvertizer/5e4ee281-95a3-442a-bb2a-5ef68cf5181a.html) is available which allows us to provide a list of allowed issuers and deny any other assignments outside of these exceptions. I would recommend using this policy which allows fine-granted control of trusted issuer and federation provider types. This should also be a preferred option over the previously mentioned (simplified) policy to block any federated credential. This policy should also be run in “audit” to check compliance of already existing issued federated and discover usage of federated identity providers before enforcing “deny”.
Enforcement of deny will only apply to any new or changed configuration of federated credentials. It will not block usage of existing credentials.
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-08-02-identify-prevent-abuse-uami-federatedcredentials/UamiFedCreds%2011.png)
    
    *Configuration of parameters on built-in initiative “[Preview]: Managed Identity Federated Credentials should be of approved types from approved federation sources”*
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-08-02-identify-prevent-abuse-uami-federatedcredentials/UamiFedCreds%2012.png)
    
    *Side Note: I was not able to configure the policy with the default provided parameters. The error message “Could not find a version of the policy set definition” appears which can be fixed by selecting the version and enable option “Include preview versions”.*
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-08-02-identify-prevent-abuse-uami-federatedcredentials/UamiFedCreds%2013.png)
    
    *“Deny” effect of Azure Policy will block adding federated credential outside of allow list by any operation to Azure Resource Manager API*
    
- **Regular review and security monitoring on issued federated credentials**
As already described, [AzADSPI](https://github.com/JulianHayward/AzADServicePrincipalInsights) gives you the opportunity to create a report of all your federated credentials and their privileges. You can include the data to Microsoft Sentinel for enrichment and correlation between security incidents and details of the configured credentials.
- **Isolate critical user assigned identities from Landing Zones**
Sensitive UAMIs should not be placed to a regular subscription. For example, UAMIs which are assigned to Azure Policies with sensitive permissions should be created and managed in a “Platform” Management Group with highly restricted access. Creating a “deny” role assignment by [Deployment Stacks on Managed Resources](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deployment-stacks?tabs=azure-powershell#protect-managed-resources) could also be an approach to block inheritance permissions.

# Summary

Independent of the previous remarks, Federated Credentials are a great and secure way to provide access to Entra ID-protected resources by trusting (authorized) workloads and 3rd party IdPs. They provide benefits over other authentication methods by reduced risks of leaked credentials, certificates expiring or maintenance. In addition, it allows to establish a kind of correlation of relationship between workload and identity. However, it should be considered, who can add the trust relationship and what is a valid and trustworthy relationship. Your governance, workload identity lifecycle processes and security policies should not only cover App Registration in Microsoft Entra ID. It is necessary to extend the view and take care of the identity resources in Microsoft Azure as well. Microsoft already provides built-in methods to identify and take control of Federated Credentials which avoids establishing access from malicious actors outside your (tenant) trust boundary. Additional efforts should be made to enrich data in your SOC for visibility of federated credentials and relation to trusted (federated) entities and subjects.