---
title: "Microsoft Entra Workload ID - Lifecycle Management and Operational Monitoring"
excerpt: "Workload identities should be covered by lifecycle management and processes to avoid identity risks such as over-privileged permissions but also inactive (stale) accounts. Regular review of the provisioned non-human identities and permissions should be part of identity operations. In this article, we will go through the different lifecycle phases and other aspects to workload identities in your Microsoft Entra environment."
header:
  overlay_image: /assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle.png
  overlay_filter: rgba(102, 102, 153, 0.85)
  teaser: /assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle.png

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
last_modified_at: 2023-08-18
---

## Inventory and initial review of Service Principals

I would recommend having a initial review for full visibility and insights of the service principals which has been already created, especially in an environment without an establish lifecycle workflow or governance framework.

[Julian Hayward](https://www.linkedin.com/in/julianhayward/) has published an awesome tool with the name “[AzADServicePrincipalInsight](https://github.com/JulianHayward/AzADServicePrincipalInsights)” (AzADSPI) which allows you to create a report across all types of workload identities. This can be also used for regular reviews and integrated to your monitoring and review process.

The report covers analyses about owner, credentials, app and directory role assignments of Application and Service Principal objects (all different types).

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-azadspinsights.png)

*AzADSPI provides a comprehensive report of application and service principal objects in your Entra ID environment*

There are many questions you should try to answer by analyzing the report, for example:

- Who has ownership to application and service principals?
    - Do you have user accounts as owners to objects with critical (classified) permissions (such as User.ReadWrite.All) or directory role assignments?
- Which objects are owned by a service principal?
- Are you aware of the existing user-assigned identities in Azure and the resources which can to use them?
- Do you trust the Identity Providers and entities which are defined in Federated Identity Credentials?
- …

Results can be exported as HTML (with visualization) but also as JSON and CSV export. Julian is also providing pre-configured pipeline files for Azure DevOps and GitHub which allows to export the report data to a repository automatically. The approach is similar what I have built with [AADOps for Conditional Access](https://www.cloud-architekt.net/aadops-conditional-access/).

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle1.png)

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle2.png)

*Pull Request in AzADSPI to compare changes of Workload Identity since latest pipeline run. In this example, a sensitive API permission and a short term client secret has been assigned.*

The tool provides an classification for API Permission with critical and medium sensitivity which can be also customized.

*Side Note: Options to integrate this tool to your Microsoft Sentinel environment (for example to build advanced logic and enrichments in analytics rules) will be one of the major topics in the next part of this blog post series. I’ve shown this approach in scenarios for enrichment of analytics rules and hunting in my community talks about Workload Identities in the recent years. Last year, I’ve started to [work on an automation for classification of privileged access](https://devblogs.microsoft.com/identity/azure-ad-recommendations-adal/) based on Microsoft’s Enterprise Access Model. This feature will be part of my “AADOps” PoC project and [covers all major RBAC systems in Microsoft Cloud](https://twitter.com/Thomas_Live/status/1688797242633699328) (Azure, Entra ID, Intune, Graph API, Identity Governance) across all types of identities (including workload identities). But this is topic for another community talks and blog posts in the near future.*

*Tip: There’s is also a tool by [Joosua Santasalo](https://github.com/jsa2/AADAppAudit) which gives you comprehensive insights to your application and workload identities with option to ingest the data to Log Analytics.* 

## Lifecycle Management of Application Objects

In the following section, I would like to evoke simply the keypoints (including a short description) what should be considered in lifecycle management from my point of view.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle3.png)

*Overview on tasks during the lifecycle phases of application objects (as an example)*

## Planning and Provisioning

### Check requirements on application-level integration

- Verify the integration of MSAL or any other authentication library which passed the following security requirements:
    - [Verify claims-based authorization](https://learn.microsoft.com/en-us/azure/active-directory/develop/access-tokens#claims-based-authorization) (logic must check security identifiers including tenant and object id)
    - Support for latest features in protocol standards and Entra ID features, such as [Conditional Access Evaluation for workload](https://learn.microsoft.com/en-us/azure/active-directory/conditional-access/concept-continuous-access-evaluation-workload) or [application identities](https://learn.microsoft.com/en-us/security/zero-trust/develop/secure-with-cae)
- Check if the app publisher offers a [publisher verification](https://learn.microsoft.com/en-us/security/zero-trust/develop/integrate-apps-with-azure-ad-microsoft-identity-platform#become-a-verified-publisher)

### Create a well-configured app registration

- Choose multi-tenant only if acceptable and consider limited security control with Conditional Access and User Assignment requirements. Those settings will be applied in the tenant of the Service Principal instance. Policies from the „home“ tenant of the app registration will not apply to users in other („resource“) tenants. In addition, you will have no sign-in events from those users.
    
    *Tip: [Merill Fernando](https://twitter.com/merill) has written a great blog post about “[Entra ID multi- vs. single tenant app](https://merill.net/2023/04/azure-ad-multi-tenant-app-vs-single-tenant-app/)” which I can strongly recommended to read.*
    
    - Consider to enable “[Application instance lock](https://learn.microsoft.com/en-us/azure/active-directory/develop/howto-configure-app-instance-property-locks)” to protect sensitive properties of the multi-tenant app in other tenants.
    
- [Follow Microsoft’s best practices](https://learn.microsoft.com/en-us/azure/active-directory/develop/security-best-practices-for-app-registration) for application configuration which covers security related aspects of Redirect URIs, authentication flow, Application and secret management.
    - [Integration assistant](https://learn.microsoft.com/en-us/azure/active-directory/manage-apps/plan-an-application-integration) in Entra ID portal should also support you in verifying the recommended configuration and go trough a checklist.
- Consider [best practices in developing Zero Trust applications](https://learn.microsoft.com/en-us/security/zero-trust/develop/overview), especially [token management](https://learn.microsoft.com/en-us/security/zero-trust/develop/token-management) (claims, lifetime, caching) are core aspects for app integration.
    - Avoid using weak claims (such as e-mail or UPN) to determine if the token subject is authorized. Validate the [subject by using immutable claim values](https://learn.microsoft.com/en-us/azure/active-directory/develop/claims-validation#validate-the-subject).
- Review and remove ownership to application and service principal objects. If needed, delegate permissions with object-scoped and least privileged directory roles (as already described in the first part of the blog post series). Consider using PIM approval flow for sensitive delegated permissions on application object (such as update credentials or change reply URLs) to implement four eyes principle and closely review all activities from eligible admin.
- Optional: Define a `notificationEmailAddresses` in the Service Principal object if you are implementing a SAML token-based application and you want deliver notification for certificate expiration.
- Optional: Use `serviceManagementReference` if you can build a link between application and service or asset by using a management ID (information will be visible for other users).

### Classification and Service Mapping of Service Principals

- Classify policy requirements for user access to the application as custom security attribute. This could include attributes related to the target audience of the app or the policy requirements to access the application:
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle4.png)
    
    The assignment of the custom security attribute allows you to use the classification in the App Filter for Conditional Access Targeting:
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle5.png)
    
- Classify application permissions of the Service Principal (e.g. sensitive Graph API Permissions) to protect them by Conditional Access for Workload Identities. I’ve started to create a [classification definition for Microsoft Graph API](https://github.com/Cloud-Architekt/AzurePrivilegedIAM/blob/main/Classification/Classification_MsGraphPermissions.json) based on the Enterprise Access Model which will be applied by an automation (as part of my PoC project “EntraOps”):
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-workbook.jpg)
    
    *Workbook of EntraOps Privileged EAM to identify highly sensitive workload identities and their privileges in Azure, Entra and to Resource Apps. API permission to “User.ReadWrite.All” will be classified to “Control plane” (Tier 0) because of the wide range of permissions to modify user accounts. The service principal needs to be protected particularly (by Conditional Access and Identity Protection for Workload Identities).*
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-classification.png)
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle6.png)
    
    *Custom security attributes will be used to “label” classification and target them in policy (for example, all control plane access should be blocked if Identity Protection has detected a high risk of the workload identity.*
    
- Require user assignment for sensitive or restricted applications to the target end-user audience. This setting is also important if an application identity has sensitive delegated permission and you want to manage the scope of users.
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle7.png)
    
    Groups can be used to assign permissions to get access to those apps or APIs. In addition, Identity Governance offers an effective way to request, manage and review user access to applications.
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle8.png)
    
    *Access to Graph Explorer and Microsoft Graph PowerShell is restricted and will be granted with access package for privileged role assignments*
    
- Use a custom security attribute or `notes` attribute to store information about service owner or application developer. I would prefer to choose a custom security attribute to keep the relation private to avoid discovery and reconnaissance by attackers.
    - Other entity relations should be also stored in custom security attributes which could be later used in Microsoft Sentinel (WatchLists) for building correlations in analytics rules, hunting queries or scoping „Conditional Access for Workload Identities“. For example: Details about hosting or running the workload (pipeline name, environment or relation other cloud resources/accounts) and classification of permissions (according to own-defined sensitivity levels or tiering model):
        
        ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle9.png)
        
        *A service principal which will be used in Azure Pipelines for sensitive operations (in this case, Entra ID automation) offers information about the classification and details on the associated pipelines in the custom security attributes.*
        

### Issuing credentials

- **Client Secret** (only if other credential types are not supported): Use KeyVault to transfer and provide secret to the workload. Avoid long lifetimes by implementing a process and way to rotate the secrets regularly. Configure a secret expiration date to take advantage of [notification trigger](https://learn.microsoft.com/en-us/azure/key-vault/general/event-grid-logicapps).
    
    There are some great blog posts by the community about client secret rotation. Check out the following articles and samples:
    
    - [How to automatically rotate Entra ID app registration client secrets using Azure Functions (with Java) and Key Vault – Jordan Bean Dev Blog](https://jordanbeandev.com/how-to-automatically-rotate-azure-ad-app-registration-client-secrets-using-azure-functions-with-java-and-key-vault/)
    - [Implement automatic key rotation on Azure DevOps service connections by Koos Goossens](https://koosg.medium.com/implement-automatic-key-rotation-on-azure-devops-service-connections-13804b92157c)
    - https://github.com/Azure/AzureAD-AppSecretManager
- **Certificate:** Use KeyVault or your trusted Certificate Authority of choice to create the private key and avoid delegating any permissions or options to export the sensitive cryptographic information. Only the public key needs to be imported to the associated app registration.
    - Choose the “new” RBAC permission model (over the classic “Vault Access Policy” permission model) to use Azure PIM and Azure Activity Logs for privileged access governance.
    - Implement a renewal process for certificates as already described in the scenario with client secrets.

    *Side Note: Applications can also roll their own existing keys. More details about how to implement it and the usage of `Application.ReadWrite.OwnedBy`  is very well described in a the blog post “[Using Application.ReadWrite.OwnedBy and addKey methods for Graph API](https://securecloud.blog/2021/12/29/using-application-readwrite-ownedby-and-addkey-methods-for-graph-api/)” by [Joosua Santasalo](https://twitter.com/SantasaloJoosua).*

- **Federated Credentials:** This credential type offers many benefits over secrets or certificates from security and operational perspective. Choose this credential type if your [workload scenario is supported](https://learn.microsoft.com/en-us/azure/active-directory/workload-identities/workload-identity-federation#supported-scenarios). A subject identifier should be chosen with a strong scope on your workload and a reliable and secure external Identity Provider (IdP) for establishing a trust relationship.

### Restrict application (credential) management

**App Instance Property Lock**

Microsoft allows to lock properties of the “Service Principal” object for Multi-Tenant Apps. This prevents admins or owners of the object in the Resource Tenant from adding credentials and impersonate the application identity. This feature is in preview and well [documented in Microsoft Learn](https://learn.microsoft.com/en-us/azure/active-directory/develop/howto-configure-app-instance-property-locks).

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle10.png)

*App Instance property lock can be configured in the “Authentication” blade of the App Registration*

### Entra ID App Management Method Policies

Client Secret and Certificate management operations can be restricted for Application and Service Principals by using App Management Policies. This feature can be only configured by using Microsoft Graph and is limited to tenants with assigned Workload Identity Premium licenses.

A tenant-wide policy (`tenantAppManagementPolicy`) can be created which applies to all apps and service principals. It’s possible to define a policy which blocks new password credentials beginning from a specific date. The properties of the policy object and schema are described in the related [resource type docs](https://learn.microsoft.com/en-us/graph/api/resources/tenantappmanagementpolicy?view=graph-rest-1.0).

```powershell
{
    "isEnabled": true,
    "applicationRestrictions": {
        "passwordCredentials": [
            {
                "restrictionType": "passwordLifetime",
                "maxLifetime": "P90D",
                "restrictForAppsCreatedAfterDateTime": "2017-01-01T10:37:00Z"
            },
            {
                "restrictionType": "symmetricKeyAddition",
                "maxLifetime": null,
                "restrictForAppsCreatedAfterDateTime": "2021-01-01T10:37:00Z"
            },
            {
                "restrictionType": "customPasswordAddition",
                "maxLifetime": null,
                "restrictForAppsCreatedAfterDateTime": "2015-01-01T10:37:00Z"
            },
            {
                "restrictionType": "symmetricKeyLifetime",
                "maxLifetime": "P40D",
                "restrictForAppsCreatedAfterDateTime": "2015-01-01T10:37:00Z"
            }
        ],
        "keyCredentials": [
            {
                "restrictionType": "asymmetricKeyLifetime",
                "maxLifetime": "P365D",
                "restrictForAppsCreatedAfterDateTime": "2015-01-01T10:37:00Z"
            }
        ]
    }
}
```

Application Management policies (`appManagementPolicy`) can be created to have specific restrictions for certain application identities and can be linked to individual application or service principal objects. These policies will override the restrictions (by the tenant-wide policy) in scope of the linked object. Details on [create, modify and link App Management policies](https://learn.microsoft.com/en-us/graph/api/resources/appmanagementpolicy?view=graph-rest-1.0) are also described in the Microsoft Graph Docs.

*Side Note: Vasil Michev has published a great blog post about this topic with the title “[Entra ID App Management Method Policies Harden Application Security Posture](https://practical365.com/azure-ad-app-management-method-policies-harden-application-security-posture/)” on Practical365.com.*

### Azure Policies for Workload Identity Federation

As far as I know, there are no options to restrict federated credentials on Application objects in Entra ID. But there is a solution if you are using user-assigned managed identities in combination with Federated Credentials. Azure Policies are the central policy engine at the level of Azure Resource Manager (ARM) as control and management plane. Uday Hegde has written an excellent blog post how to [create and apply a policy to restrict federation with an approved set of issuers.](https://blog.identitydigest.com/azuread-mi-federate-policy/)

## Operational Monitoring and Maintenance

There are a couple of sources and signals in Entra ID but also Microsoft 365 Defender and Microsoft Entra product family which should be included in the operational monitoring.

### Entra ID Recommendations on Workload Identities

The following checks are included in the recommendations blade and are available in Entra ID P2 tenants:

- [Remove unused applications](https://learn.microsoft.com/en-us/azure/active-directory/reports-monitoring/recommendation-remove-unused-apps)
(Microsoft.Identity.IAM.Insights.StaleApps)
- [Remove unused credentials from applications](https://learn.microsoft.com/en-us/azure/active-directory/reports-monitoring/recommendation-remove-unused-credential-from-apps)
(Microsoft.Identity.IAM.Insights.StaleAppCreds)
- [Renew expiring application credentials](https://learn.microsoft.com/en-us/azure/active-directory/reports-monitoring/recommendation-renew-expiring-application-credential) (Microsoft.Identity.IAM.Insights.ApplicationCredentialExpiry)
- [Renew expiring service principal credentials](https://learn.microsoft.com/en-us/azure/active-directory/reports-monitoring/recommendation-renew-expiring-service-principal-credential)
(Microsoft.Identity.IAM.Insights.ServicePrincipalKeyExpiry)

You’ll find the overview of recommendations in the [Entra ID portal](https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/Overview) and as [deep link in the “Workload Identities” blade](https://entra.microsoft.com/#view/Microsoft_Azure_ManagedServiceIdentity/WorkloadIdentitiesBlade) from the Microsoft Entra portal:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle11.png)

*Overview of Workload Identities in the Microsoft Entra portal*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle12.png)

*Recommendations covers a few checks for app registrations/service principals including unused permissions and credentials.*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle13.png)

*Details on impacted resources and remediation steps are available for every recommendation. Status will be automatically updated if the application or service principals have been modified according to the action plan. You can also change the status manually (active, dismissed or postponed).*

All recommendations can be [listed and modified by Microsoft Graph API](https://learn.microsoft.com/en-us/graph/api/resources/recommendations-api-overview?view=graph-rest-beta#types-of-recommendations). This allows you to implement the insights to your existing operational monitoring or dashboard solution.

Using a filter on `ImpactType` give us the option to get only findings regarding resource types “Applications” which also includes Service Principals:

```powershell
https://graph.microsoft.com/beta/directory/recommendations?$filter=impactType eq 'apps'
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle14.png)

*Programmatically access to the recommendation by using Microsoft Graph API by using Graph Explorer*

As you can see in the previous screenshot, the impacted resources are missing. Another API call allows to [get a list of the related entities](https://learn.microsoft.com/en-us/graph/api/impactedresource-get?view=graph-rest-beta&tabs=http). But you need to include the full ID of the recommendation, including the specific “Tenant Id” and “Resource Name” of the recommendation: 

```powershell
https://graph.microsoft.com/beta/directory/recommendations/<TenantId>_Microsoft.Identity.IAM.Insights.ApplicationCredentialExpiry/impactedResources
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle15.png)

*Details on the impacted resource (service principal or application) from the recommendation.*

### Usage & Insights about sign-in activities and credentials

Entra ID provides integrated activity reports in the “[Usage & Insights](https://learn.microsoft.com/en-us/azure/active-directory/reports-monitoring/concept-usage-insights-report)” blade.
Furthermore, the results of these reports are also accessible from Microsoft Graph API and can be integrated in your monitoring solution. The following activity reports are particularly related to workload identities.

**Service principal sign-in activity**

Last sign-ins (with Time Stamp and Request Id) from app-only (application permissions) or user access (delegated permissions) will be covered in the report. `lastSignInRequestId`  can be used for searching the related user or service principal sign-in in the Entra ID sign-logs.

```powershell
https://graph.microsoft.com/beta/reports/servicePrincipalSignInActivities
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle16.png)

**Entra ID application activity**

This report shows a summary of all users’ sign-in attempts to an application including error codes.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle17.png)

*Report displays the error description of the sign-in failures and counts and timeline of all user sign-ins*

The API endpoint “[applicationSignInSummary](https://learn.microsoft.com/en-us/graph/api/resources/applicationsigninsummary?view=graph-rest-beta)” allows to get a summary with counts on successful/failed sign-ins and success percentage filtered by a pre-defined period of time.

```powershell
https://graph.microsoft.com/beta/reports/getAzureADApplicationSignInSummary(period='D7')
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle18.png)

*Summary of application sign-in report within the last 30 days (maximum time range).*

Another API call to “[applicationSignInDetailedSummary](https://learn.microsoft.com/en-us/graph/api/resources/applicationsignindetailedsummary?view=graph-rest-beta)” is needed if you are interested to get all details about the sign-in counts and failures:

```powershell
https://graph.microsoft.com/beta/reports/applicationSignInDetailedSummary
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle19.png)

The response shows the single records of the report which will be aggregated regularly and includes additional details about the `failureReason` .

**Application credential activity**

Monitoring to expiring of client secrets and certificates is one of the essential operational tasks during the maintenance phase of application and workload identities. The following reports give a detailed overview of expiring credentials.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle20.png)

The Portal UI allows you to filter for the expiration time window, certificate types and recent sign-in activities. This feature is in preview and I’ve running into some issues with outdated data and filter.

All details of the report can be also listed by using the following Graph API call:

```powershell
https://graph.microsoft.com/beta/reports/appCredentialSignInActivities
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle21.png)

*Credential activity report covers not only expiration of specific credentials, but it’s also shows the latest sign-in activity. Unfortunately, there are some issues with the quality of data, as you can see in the screenshot (example: `lastSignInDateTime`*)

### Entra ID Workbooks and Alerts for advanced operational monitoring

Sign-in of users (`SigninLogs` and `NonInteractiveUserSignInLogs`) but also service principals (`ServicePrincipalSignInLogs`) are covered by Entra ID and can be forwarded to a Log Analytics or Microsoft Sentinel workspace by using [Diagnostic settings](https://learn.microsoft.com/en-us/azure/active-directory/reports-monitoring/howto-integrate-activity-logs-with-log-analytics).

The following examples should give you an overview about the capabilities and options to use this data to visualize insights about your integrated apps.

**App sign-in health**

This is an integrated workbook from the Entra ID monitoring blade and visualizes the numbers of successful sign-ins and failures (like previous described Usage & Insights report “Application Activity”). But it offers longer time ranges (based on your workspace data retention), customized views and an overall status.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle22.png)

*Tip: Error Codes will be only displayed in some scenarios of troubleshooting sign-in issues. Check out the references of [AADSTS error codes](https://learn.microsoft.com/en-us/azure/active-directory/develop/reference-error-codes#aadsts-error-codes) but also the [integrated lookup tool for resolving the code numbers](https://learn.microsoft.com/en-us/azure/active-directory/develop/reference-error-codes#lookup-current-error-code-information).*

### Analyses of used Authentication Library

I’ve built a [KQL query](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Hunting%20Queries/AAD-WorkloadIdentities/AuthenticationLibraries.kusto) which is parsing the information about the implemented Authentication Library from the `AADServicePrincipalSignInLogs` sign-in logs.

This should help to identify outdated versions from Microsoft Authentication Library (MSAL) or legacy libraries (e.g., ADAL):

In my opinion, you should use the latest version to avoid security vulnerabilities, known issues with core features (such as token caching) and missing features (e.g., support for CAE).
The query can be also used for visualization as you can see in the following sample:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle23.png)

*Side Note: Recently, Microsoft has announced a check in “Entra ID Recommendation” for [identifying ADAL Applications](https://devblogs.microsoft.com/identity/azure-ad-recommendations-adal/). Keep this in mind, if you are looking for implementations of this deprecated Authentication Library.*

### Issued CAE token

Continuous access evaluation (CAE) is also available for workload identities. But how to detect which non-human identity or session is using a CAE-capable token?

You can get insights about this one in the Entra ID sign-in blade by filtering on “Is CAE token” and check the “Additional Details” tab:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle24.png)

Unfortunately, the details if CAE token has been issued are not available in the Diagnostic Logs of `AADServicePrincipalSignInLogs` yet. 

### Governance of OAuth Apps and Permissions

Microsoft has been released “App Governance” as add-on feature for “Microsoft Cloud App Security” (now called “Microsoft Defender for Cloud Apps”) in Summer 2021. An additional licensing was required but this [will be change for Microsoft 365 E5 and E5 Security customers](https://techcommunity.microsoft.com/t5/microsoft-365-defender-blog/rsa-news-taking-xdr-for-saas-apps-to-the-next-level-app/ba-p/3804722) on June 1st, 2023. The feature will be available as opt-in at no additional costs.

This tool gives you an comprehensive overview and insights about your applications in different areas (data and permission usage, access to sensitive data or delegated access by sensitive accounts).

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle25.png)

“*App Governance” is fully integrated to “Microsoft 365 Defender” portal and shows many compliance & security related insights.*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle26.png)

Summary of applications gives you an overview about certification and publisher verification which should be important to review for multi-tenant and SaaS applications. But also, deep links to related “Entra ID recommendations” have been integrated.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle27.png)

*Policy templates but also custom policies can be configured for monitoring and detection of different scenarios. Alerts for “unused apps”, “unused credentials” or “expiring credentials” are available as well. So, there are some overlapping features between Entra ID recommendations and App Governance.*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle28.png)

*Alert integration to Microsoft 365 Defender/Microsoft Sentinel, custom app scope and automated response (disable app) could be one of the reason to prefer the features in App Governance over Entra ID recommendations. More details about [App hygiene policies](https://learn.microsoft.com/en-us/defender-cloud-apps/app-governance-secure-apps-app-hygiene-features) are available from Microsoft Learn.*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle29.png)

*Statistics about data usage of an app to OneDrive (via Microsoft Graph API) and  overview about users which has been defined as priority/sensitive account.*

More policies and capabilities for detecting anomalous activities are available which will be described in the next part of the blog post about “Monitoring and Security of Entra ID Workload Identities”.

### Remove unused permissions

Regular review of assigned permissions should be considered for workload identities. App Governance has a strong focus on permissions to API Permissions. But also, other privileges to RBAC assignments (such as Entra ID roles or Azure RBAC) and even Groups should be included in the access review.

**MDA App Governance for Microsoft API Permissions**

App Governance gives you the option to analyze the usage of Graph API Permissions for Exchange Online, SharePoint, OneDrive and Teams in the recent 90 days. This can be also integrated as a policy to trigger an alert but also for correlation to other built-in threat detections (”[Increase in data usage by an overprivileged or highly privileged app](https://learn.microsoft.com/en-us/defender-cloud-apps/app-governance-investigate-predefined-policies#increase-in-data-usage-by-an-overprivileged-or-highly-privileged-app)”).

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle30.png)

*Overview of Assigned Permissions to Exchange Online (Mail.ReadWrite) and User.Read (Entra ID). “In Use” can be only analyzed for certain endpoints of Microsoft 365 services.*

*Side Note: Microsoft seems to be working on a new data source in Entra ID logs to get insights about Microsoft Graph Activities. There are some reports about the [private preview on Twitter](https://twitter.com/DrAzureAD/status/1646396172804784129). The [log schema is already available in Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/microsoftgraphactivitylogs) and gives some interesting impressions which information will be covered. In my opinion, this would also allow to provide detections in Microsoft Sentinel for unused/over-privileged API permissions but detecting abuse and exfiltration of data from/to Graph.*

**Entra Permissions Management (EPM) for Multi-Cloud Permissions**

Over-privileged permissions in cloud infrastructure environments can be analyzed with Entra Permissions Management (EPM). Currently, Google Cloud Platform (GCP), Amazon Web Services (AWS) and Azure are supported. A score of unused or excessive permissions will be calculated with the option to create a custom role assignment based on the used and required permissions.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle31.png)

*Example: Microsoft Sentinel Playbook is running with a system-assigned managed identity which has comprehensive permissions as part of the role assignment to “Microsoft Sentinel Contributor”. But only one role action (“incidents/comments/write”) will be used from the role definition set of over 700 actions. Reducing the set of assigned permissions can be achieved by the remediation features in EPM and supports you to follow the approach of least privileges.*

I can strongly recommend evaluating EPM in your environment by using the free trial version. There’s also a comprehensive “Trial user guide” which supports you to explore all the features to analyze least privilege in your multi-cloud environment.

**Access Review for Entra ID roles and Azure resource assignments**

Identity Governance supports access review for service principal role assignments in Entra ID or Azure Resources. This feature has been already [introduced in June 2021](https://techcommunity.microsoft.com/t5/microsoft-entra-azure-ad-blog/introducing-azure-ad-access-reviews-for-service-principals/ba-p/1942488) and requires “Workload Identity Premium License” in addition to Entra ID Premium P2.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle32.png)

*Configuration of Access Review for Service Principals for high-privileged directory roles.*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle33.png)

*Automated actions can be configured in case the reviewer doesn’t respond to confirm the need of the required permissions.*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle34.png)

*Reviewer will be notified about the access review and needs to approve or deny the continuing validity of the requirements to use this directory or RBAC role assignment. There seems to be no support for “recommended actions” which gives insights about the current usage. A deep link to the recent activities as part of the Azure (AD) Audit Logs is available from the review page.*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle35.png)

*Privileged Identity Management (PIM) gives you a total number of directory role assignments. Unfortunately, assignments on Administrative Unit- or Object-Level have not been recognized in my test environment.*

### Recovery of deleted objects

Entra ID offers an option to recover supported objects within a 30-day time window. Those objects will not be permanently deleted and remains in a suspended state for this time period. App Registrations are one of these supported objects which can be [restored when they have been removed recently](https://learn.microsoft.com/en-us/azure/active-directory/develop/howto-restore-app). Managed Identities are a special type of service principals and are not covered. Some of the configurations can not be recovered from the Portal UI or not included in the recovery process yet. Check the [deletion and recovery FAQ](https://learn.microsoft.com/en-us/azure/active-directory/manage-apps/delete-recover-faq) for more details.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-08-18-entra-workload-id-lifecycle-management-monitoring/workloadid-lifecycle36.png)

*You’ll find the options to delete an application permanently or recover the object by clicking on the “Deleted application” tab on the “App Registration” blade.*

## Summary and comparison of workload identity lifecycle management

|  | Application Identity (with Key or Certificate) | Application Identity (with Federated Credential) | Managed Identity (System-Assigned) | Managed Identity (User-Assigned) |
| --- | --- | --- | --- | --- |
| Use Cases | No limitations | Limited, support workload and/or IdP required | Limited, support, workload must be a Azure-Managed resource | Limited, support, workload must be a Azure-Managed resource or federated credential |
| Security Boundary | Single- or Multi-Tenant | Single- or Multi-Tenant | Single Tenant, limited Multi-Tenant access* | Single-Tenant, limited Multi-Tenant access* |
| Recovery Options | Soft Deletion | Soft Deletion | N/A | N/A |
| Lifecycle Management | Managed by admin or automated process | Managed by admin or automated process | Managed by Azure  | Standalone Azure resource (managed by admin or automated process) |
| Recovery Options | Soft Deletion | Soft Deletion | N/A | N/A |
| Token Lifetime / Cache | 1h (Default), 24h (CAE) | less than or equal to 1h | [up to 24h](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/managed-identity-best-practice-recommendations#limitation-of-using-managed-identities-for-authorization) |   [up to 24h](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/managed-identity-best-practice-recommendations#limitation-of-using-managed-identities-for-authorization) |
| Delegation and Ownership | Application/Enterprise App Owner Entra ID Role (Directory, Object) | Application/Enterprise App Owner, Entra ID Role (Directory, Object) | Enterprise App Owner, Entra ID Role, Azure RBAC Role/Resource Owner | Enterprise App Owner, Entra ID Role, Azure RBAC Role/Resource Owner |
| Recovery Options | Soft Deletion | Soft Deletion | N/A | N/A |

*Next: Advanced Monitoring and Security*

*I’ve already mentioned in this part of the blog post series some Microsoft Security but also community tools for monitoring. In the next part we will go into details about using the capabilities of this solutions for detection and response of workload identities. Estimated publication date date is around end of September/Early October.*