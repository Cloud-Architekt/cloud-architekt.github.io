---
title: "Microsoft Entra Workload ID - Incident Response with Microsoft Sentinel Playbooks and Conditional Access"
excerpt: "In the recent parts of the blog post series, we have gone through the various capabilities to detect threats and fine-tune incident enrichment of Workload Identities in Microsoft Entra. This time, we will start to automate the incident response for tackling malicious activities and threats. This includes the usage of Conditional Access for Workload ID but also configuring a Microsoft Sentinel Playbook with the least privileges."
header:
  overlay_image: /assets/images/2024-01-12-entra-workload-id-incident-response/workloadidincidentresponse.png
  overlay_filter: rgba(102, 102, 153, 0.85)
  teaser: /assets/images/2024-01-12-entra-workload-id-incident-response/workloadidincidentresponse.png

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
last_modified_at: 2024-01-12
---

*Cover image: Content credentials Generated with AI (generated by Microsoft Bing Image Creator), January 12, 2024 at 7:48 AM*

## Incident Response Playbook templates

Microsoft has been released playbooks for security incident scenarios of [Compromised and malicious applications investigation](https://learn.microsoft.com/en-us/security/operations/incident-response-playbook-compromised-malicious-app) but also for [App consent grant investigation](https://learn.microsoft.com/en-us/security/operations/incident-response-playbook-app-consent).

In case of a true positive alert, the workload identity should be blocked for further malicious activity. In general, there are two different options to stop further sign-in activity by using Portal UI or Microsoft Graph API:

- [Disable service principal](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/disable-user-sign-in-portal?pivots=portal)
- [Confirm service principal compromised](https://learn.microsoft.com/en-us/entra/id-protection/concept-workload-identity-risk#investigate-risky-workload-identities) *(requires Workload Identity Premium licenses)*

Both events are also supported revocation events for [Continuous access evaluation for workload identities](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation-workload) (CAE). This allows to revoke existing access tokens if the application/workload and resource provider a,
,re supporting CAE. Otherwise, the access token is still valid and further granted access will be valid until the token expires. 

*Side Note: CAE does not support Managed Identities and also the support for resource providers are limited (e.g., Microsoft Graph).*

### Playbook: Confirm compromised (risky) Service Principal

Let’s have a closer look t how we can trigger a playbook from an incident successfully.
First, we need to make sure that the `ObjectId` of the Service Principal is available by using the trigger of Microsoft Sentinel Incidents. Both Graph API calls (Disable and confirm compromise) require the `ServicePrincipalObjectId` for the request. We have already defined and considered to establishing a clean entity mapping in the various Analytics Rules which we configured in the previous parts of the blog post. However, Microsoft Sentinel offers only an entity mapping to the `AppId` of in the incident.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-01-12-entra-workload-id-incident-response/workloadidautoir.png)

Unfortunately, the Application Id was also not available in the Outputs of the Incident Playbook Trigger during my tests:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-01-12-entra-workload-id-incident-response/workloadidautoir1.png)

Therefore, I’ve decided to implement the following logic to get the custom alert details from each alert after the Sentinel Incident has been triggered. We have used this option before to enrich the alert with details from the `WorkloadIdentityInfo`.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-01-12-entra-workload-id-incident-response/workloadidautoir2.png)

Next, the `ServicePrincipalId` from the “Custom Alert Details” will check whether the value is not empty. Additional “Custom Alert” fields can be also used in the condition, for example if the enriched information shows the application for `PrivilegedAccess` or as `IsFirstPartyApp` . This allows you to run the playbook on a particular scope only.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-01-12-entra-workload-id-incident-response/workloadidautoir3.png)

Afterwards the Graph API call will execute to mark the Workload Identity as compromised. The HTTP action will be used with the associated (system-assigned) Managed Identity and the required permissions (`IdentityRiskyServicePrincipal.ReadWrite.All`). The sign-in will be blocked if a corresponding Conditional Access policy for Workload Identity has been configured.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-01-12-entra-workload-id-incident-response/workloadidautoir4.png)

Finally, a comment will be added to the Incident if the workload identity has been marked as compromised. This requires that the Logic App has permission to the Azure RBAC role “Microsoft Sentinel Responder” or least privileges on the specific role action to write incident comments (`[Microsoft.SecurityInsights](https://learn.microsoft.com/en-us/azure/role-based-access-control/resource-provider-operations#microsoftsecurityinsights)/incidents/`).

The ARM deployment file of the Logic App can be found here:

[Confirm-RiskyEntraWorkloadId_Deploy.json](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Playbooks/EID-WorkloadIdentities/Confirm-RiskyEntraWorkloadId_Deploy.json)

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FCloud-Architekt%2FAzureSentinel%2Fmain%2FPlaybooks%2FEID-WorkloadIdentities%2FConfirm-RiskyEntraWorkloadId_Deploy.json)

*Keep in mind, this is just an example without any warranty. Additional enhancements are needed to run this in production, such as error handling if Graph API isn’t successful or invalid `ServicePrincipalObjectId` in Custom Details*

*Recommended to read: [Derk van der Woude](https://twitter.com/DerkVanDerWoude) has written a great blog post [how to send a alert e-mail notification to the app owner](https://derkvanderwoude.medium.com/azure-ad-identity-protection-risky-workload-alert-e-mail-notification-c6a210a79bd4) when a risky workload has been detected. This is a very interesting scenario if you want to integrate a notification option in the playbook.*

### Playbook: Disable Service Principal

A similar playbook can be also created to execute the following Graph API call to disable the workload identity.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-01-12-entra-workload-id-incident-response/workloadidautoir5.png)

But this Graph API call requires to assign `Application.ReadWrite.All` application permissions to the Managed Identity of the playbook which is a highly sensitive permissions (includes credential management for every workload identity). Therefore, I’ve created a custom directory role which includes only the required action `[microsoft.directory/servicePrincipals/disable](http://microsoft.directory/servicePrincipals/disable)`. This custom role “Workload Identity Security Responder” can be also granted to particular service principals instead of assigning to directory-level.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-01-12-entra-workload-id-incident-response/workloadidautoir6.png)

Don’t forget to include also the permission `[microsoft.directory/servicePrincipals/enable](http://microsoft.directory/servicePrincipals/disable)` for creating also playbook if the service principal should be re-enabled.

The playbook to disable a service principal can be deployed by using this ARM Template:
[Disable-EntraWorkloadId_Deploy.json](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Playbooks/EID-WorkloadIdentities/Disable-EntraWorkloadId_Deploy.json)
[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FCloud-Architekt%2FAzureSentinel%2Fmain%2FPlaybooks%2FEID-WorkloadIdentities%2FDisable-EntraWorkloadId_Deploy.json)


## Automated Response on Risky Workload ID with Conditional Access

Customers with Workload Identity Premium licenses can use Conditional Access for Single-Tenant Service Principals. Blocking access for risky service principals is one of the supported conditions and controls. I have used “Custom Security Attributes” to add the defined classification of privileged access and associated service (CMDB connection) on the Enterprise Application object. 

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-01-12-entra-workload-id-incident-response/workloadidautoir7.png)

Using those attributes has several advantages (in my opinion):

1. This information [can not be read by any member](https://learn.microsoft.com/en-us/entra/fundamentals/custom-security-attributes-overview#why-use-custom-security-attributes) in the tenant and they can be used for [App Filtering in Conditional Access Policies](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-filter-for-applications). 
2. Using filter in Enterprise Application blade to find workload identities with classification on specific Privileged Access Tiering Level.
3. Available data source in Microsoft Graph with scoped read- and write-permissions to store answer the following questions: What should the workload identity be used for (defined in onboarding process) and what are the privileges that has been assigned?

In this example, I’ve created a policy which should block access for every risky workload identity (on risk level “high”) with privileges on “ControlPlane” and that will be used for Entra ID automation.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-01-12-entra-workload-id-incident-response/workloadidautoir8.png)

There are also some other scenarios to use the attributes to include or exclude workload identities from automated response. For example, high-sensitive workloads which are covered by a policy to block any access “in the case of imminent danger”.

## Applied Incident Response for CAE-supported Workload

In the following tests, I’m using a sample from the “[ms-identity-dotnetcore-daemon-graph-cae](https://github.com/Azure-Samples/ms-identity-dotnetcore-daemon-graph-cae)” repository on GitHub to run a simple daemon console application for requesting Graph queries.
The sample application and the resource (Graph API) are supporting CAE.

This is also visible from the Sign-in logs of Entra ID but seems not included in the `AADServicePrincipalSignInLogs` in Microsoft Sentinel. 

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-01-12-entra-workload-id-incident-response/workloadidautoir9.png)

The sample application uses a service principal which has been confirmed as compromised by the previously described Sentinel Playbook which raised the risk level to “high”. As we can see in the following `AADRiskyServicePrincipals` logs, the risk has been set on 2:22 PM (UTC+1).

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-01-12-entra-workload-id-incident-response/workloadidautoir10.png)

After a few minutes, the continuously running job (query and modifying role assignments) has been stopped even though the access token (acquired from MSAL cache) was not expired yet.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-01-12-entra-workload-id-incident-response/workloadidautoir11.png)

The related `AADRiskyServicePrincipals` event entry shows that Conditional Access has blocked the access after re-evaluation of the policy (condition has changed from risk level none to high).

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2024-01-12-entra-workload-id-incident-response/workloadidautoir12.png)

## Licensing and comparison for Workload Identities features

As already described, some of the features requires additional licenses. Below you’ll find a quick overview about Entra ID features which has been included in this article but also the previous part of the blog post series.

| Feature | Free (Azure AD License) | Premium ($3/Month/Service Principal) |
| --- | --- | --- |
| Authentication, Authorization, Sign-in and Audit Logs | Yes | Yes |
| Conditional Access | No | Yes |
| Access Reviews | No | Yes |
| Identity Protection | No | Yes |
| App Management Policies | No | Yes |
| MDA App Governance | Add-on to MDA, free for E5 customers (starting June 1st, 2023) | Not Included |
| Entra Permission Management | Standalone product, $125 per resource, per year | Not Included |

In my opinion, Workload Identities Premium offers some interesting features with Conditional Access and ID Protection. Investments in implementing analytics rules, creating playbooks and fine-tune Microsoft Defender XDR alerts are highly recommended and gives you a great opportunity to build advanced and enriched detections, in addition to Microsoft’s Threat Intelligence.
