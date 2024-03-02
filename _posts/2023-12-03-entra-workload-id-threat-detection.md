---
title: "Microsoft Entra Workload ID - Threat detection with Microsoft Defender XDR and Sentinel"
excerpt: "Attack techniques has shown that service principals will be used for initial and persistent access to create a backdoor in Microsoft Entra ID. This has been used, for example as part of the NOBELIUM attack path. Abuse of privileged Workload identities for exfiltration and privilege escalation are just another further steps in such attack scenarios. In this part, we will have a closer look on monitoring workload identities with Identity Threat Detection Response (ITDR) by Microsoft Defender XDR, Microsoft Entra ID Protection and Microsoft Sentinel."
header:
  overlay_image: /assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidthreatdetection.png
  overlay_filter: rgba(102, 102, 153, 0.85)
  teaser: /assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidthreatdetection.png

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
last_modified_at: 2023-12-03
---

This blog post is part of a series about Microsoft Entra Workload ID:
- [Introduction and Delegated Permissions](https://www.cloud-architekt.net/entra-workload-id-introduction-and-delegation)
- [Lifecycle Management and Operational Monitoring](https://www.cloud-architekt.net/entra-workload-id-lifecycle-management-monitoring/)
- [Threat detection with Microsoft Defender XDR and Sentinel](https://www.cloud-architekt.net/entra-workload-id-threat-detection)
- [Advanced Detection and Enrichment in Microsoft Sentinel](https://www.cloud-architekt.net/entra-workload-id-advanced-detection-enrichment)
- [Incident Response](https://www.cloud-architekt.net/entra-workload-id-incident-response/)

## Overview of techniques and tactics

There are a couple of ways a Workload ID can be used in attack paths. They have become popular for attackers because of weakness in security configuration and monitoring:

- Limited coverage by Security Operations
- Limited protection capabilities, compared to privileged human identities
- Stale or unsecured configurations (including credential and lifecycle management)
- Delegation and usage to lower protected users (e.g., service owners or developers)

As we can see in the following example from “Solorigate” attacks, service principals have been used for (persistent) access to exfiltrate data from the Microsoft Graph API.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon0.png)

*In the past, “[Solorigate](https://techcommunity.microsoft.com/t5/microsoft-entra-azure-ad-blog/understanding-quot-solorigate-quot-s-identity-iocs-for-identity/ba-p/2007610)” was one of the known attack paths which used an existing privileged application to gain access to sensitive data.
Image Source: [Microsoft TechCommunity "Solarigate"'s Identity IoCs](https://techcommunity.microsoft.com/t5/microsoft-entra-azure-ad-blog/understanding-quot-solorigate-quot-s-identity-iocs-for-identity/ba-p/2007610)*

### Example of (multi-stage) attack path and relation to MITRE TTPs

Let us have a closer look into an attack path by abusing privileged Workload ID by compromised (lower privileged user) owner. In most scenarios, a valid and legitimate Workload ID has been taken over by an attacker. MITRE ATT&CK framework covers a few techniques which are in relation to this scenario.

1. [Valid Accounts: Cloud Accounts, Sub-technique T1078.004](https://attack.mitre.org/techniques/T1078/004/):
Compromising a user with assigned privileges by Entra ID role to manage Workload IDs or delegated ownership of a single application or service principal object. Account takeover of a user with access to the workload or DevOps environment which runs or executes on behalf of the service principal can be another option.
2. Afterwards, it would be the goal of an attacker to take over control of a non-human identity to avoid satisfying security controls for privileged user (MFA, PIM, Device Compliance, and other policies which are not applicable or applied to service principals). The following different techniques are covered by MITRE and just a few examples about used techniques which allows attacker to gain access to impersonate the application or workload:
    1. [Account Manipulation: Additional Cloud Credentials, Sub-technique T1098.001](https://attack.mitre.org/techniques/T1098/001/)
    Compromised identity has privileges to add credentials to existing application or workload identity. Additional Azure Service Principal Credentials are explicitly described in [MITRE D3FEND](https://d3fend.mitre.org/offensive-technique/attack/T1098.001/). 
    2. [Steal Application Access Token, Technique T1528](https://attack.mitre.org/techniques/T1528/)
    Access to resources or code which will be used to acquisite tokens for Workload Identity (e.g., executed pipeline in a environment with a valid federated trust to an Entra ID Federated Credential).
    3. [Unsecured Credentials: Private Keys, Sub-technique T1552.004](https://attack.mitre.org/techniques/T1552/004/)
    Credentials or private keys for Workload ID are insecurely stored, such as code artifacts or configuration files.
    4. [Gather Victim Identity Information: Credentials, Sub-technique T1589.001](https://attack.mitre.org/techniques/T1589/001/)
    Credentials are leaked to the public (darkweb or public repository)
3. [Collection, Tactic TA0009](https://attack.mitre.org/tactics/TA0009/)
Workload ID will be used for one of the techniques to gather sensitive information for the goal of this attack, such as exploring vulnerabilities for further attacks or finding resources for data exfiltration.
4. Finally, various techniques can be used as a next step for gaining access to data or resources but also further privilege escalation. For example, abusing `Mail.Read.All` Graph API Permissions for [Email Collection, Technique T1114](https://attack.mitre.org/techniques/T1114/) or using granted access to subscription for Crypto mining by [Resource Hijacking - Technique T1496](https://attack.mitre.org/techniques/T1496/).

There are also a couple of other attack scenarios in relation to application identities, for example create a malicious Workload ID by Illicit Consent Grant ([User Execution, Technique T1204](https://attack.mitre.org/techniques/T1204/)). More details can be found in the [Azure AD Attack & Defense Playbook.](https://github.com/Cloud-Architekt/AzureAD-Attack-Defense/blob/main/ConsentGrant.md)

### Security consideration on different types of workload identity

Several types of Workload Identities are available with different capabilities and support for security features, as already described in the first part of the blog post.

Below you will find a short comparison of the application and managed identity types.

|  | Application Identity (with Key- or Certificate) | Application identity (with Federated Credentials) | Managed Identity for Azure Resources |
| --- | --- | --- | --- |
| Security Boundary | Single- or multi-tenant | Single- or multi-tenant | Single-tenant* |
| Delegated  Management | Application/Enterprise App Owner, Enterprise App Owner, Entra ID role | Application/Enterprise App Owner, Enterprise App Owner, Entra ID role | Entra ID role on Directory or Object-level Azure RBAC Role/Resource Owner |
| Security Dependencies | Secure storing of credentials, Protection of App Reg/Service Principal object | Security of Federated Workload/IdP, Protection of App Reg/SP object | Security and restricted management of Azure Resource(s) and SP object |
| Restrict token acquisition | Conditional Access (Single Tenant only) | Conditional Access (Single Tenant only) | Not Available |
| Detection for Identity Attacks | Identity Protection, Sign-in logs | Identity Protection, Correlation between Entra ID and Trusted IdP AuthN/AuthZ logs | Limited Sign-in logs |
Response time to invalid issued token | 1h (Default), Few minutes when CAE is supported | 1h (Default), Few minutes when CAE is supported | [24h (by design)](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/managed-identity-best-practice-recommendations#limitation-of-using-managed-identities-for-authorization), No support for CAE |

_*Assigned permissions to other tenants via Microsoft Lighthouse delegation_

### No built-in protection by assigned (sensitive) roles and permissions

Human identities (user accounts) with [assignment to Entra ID roles or role-assignable groups are particular protected](https://www.cloud-architekt.net/restricted-management-administrative-unit/#overview-of-protection-and-delegation-capabilities-by-using-rmau-and-role-assignable-groups) by Microsoft Entra. This is not the case for Service Principals. Furthermore, Restricted Management AUs cannot be used to prevent administrative access from directory-level roles in Entra ID. It’s important to keep in mind!

## Identity Threat Detection and Response (ITDR) with Microsoft Security

There are a couple of general approaches which can be used as signal to detect suspicious sign-in or access for identities in Microsoft Entra. Let us have a look on some examples and try to map and evaluate them to the scenarios of Workload Identities

- **Repeating Patterns for timing, behavior, and target resources**
Most workloads are using Service Principals for recurring tasks or executing processes on a defined scope.
- **Identified source of access (IP address, resource)**
Partially suitable, works for resources with dedicated connectivity which can be clearly assigned,  most cloud-based PaaS services are using public endpoints which will be shared with other services.
- **User Agents or identifier of client app**
Partially suitable, works only as indicator because it’s easily to fake

All the previously named signals can be involved to build anomaly detection model and behavior analytics for non-human identities. In the next section, we will go through various data sources and features which can be used to implement detections.

### Microsoft Defender XDR Alerts and Behaviors

In the past, threat detection alerts for various OAuth app activities have been generated byMicrosoft Defender for Cloud Apps (MDA). Microsoft has decided to move some of the alerts with a high false positive rate to the new category of behaviors events.

> To enhance the quality of alerts generated by Defender for Cloud Apps, and lower the number of false positives, Defender for Cloud Apps is currently transitioning security content from *alerts* to *behaviors*.
> 

More details about Defender for Cloud Apps' transition from alerts to behaviors can be found at [Microsoft Learn](https://learn.microsoft.com/en-us/defender-cloud-apps/behaviors#defender-for-cloud-apps-transition-from-alerts-to-behaviors).

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon1.png)

*Some of the anomaly detections has been disabled as alert, as we can see in the “policies” in the Microsoft 365 Defender portal.*

Nevertheless, some detections (such as “Unusual ISP for an OAuth App”) are still available as enabled “Threat detection” policy and should be considered (in my opinion) for your ITDR implementation. In this example, I have replayed a token from an Azure Managed Identity which leads to the following (true-positive) alert.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon2.png)

Usage Pattern of access from Workload ID (in this case a Managed Identity) was involved in this alert to identify a potential compromise. An incident in Microsoft Sentinel will be generated when using the Microsoft 365 Defender Data Connector.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon3.png)

The shown incident is available in Microsoft Sentinel and displays related IP address but not the impacted Service Principal as Entity. Similar incidents from other data sources works (based on IP address entity) and allows to show context to other detections in Microsoft Defender for Cloud (”MicroBurst exploration toolkit used”) but also similar alerts from Microsoft Sentinel Analytics Rules.

The required information for entity mapping to the OAuth application is included in the `SecurityAlert` entry and could be used for building an own mapping by implementing an Analytic Rule in Microsoft Sentinel.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon4.png)

*Details of the Service Principal (oauth-application) are included in the entities even if they are considered by Microsoft Sentinel for entity mapping.*

Some other detections has been moved to Behaviors (e.g., “Unusual addition of credentials to an OAuth app”) and are documented in the [associated Microsoft Learn article](https://learn.microsoft.com/en-us/defender-cloud-apps/behaviors#supported-detections).

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon5.png)

*Advanced hunting allows us to get details of the behavior detections (from the table `BehaviorInfo`) including enriched information from  "App Governance" about the Entra ID application (formerly known as [OAuth app inventory](https://learn.microsoft.com/en-us/defender-cloud-apps/manage-app-permissions) in MDA). This includes also to getting a list of all delegated or application permissions.*

```jsx
BehaviorInfo
| where ActionType == "UnusualAdditionOfCredentialsToAnOauthApp"
| join BehaviorEntities on BehaviorId
| where EntityType == "OAuthApplication"
| extend Permissions = parse_json(AdditionalFields1).Permissions
```

A custom detection can be created (by using the button named “Create detection rule”) right from the “Advanced hunting” page. This allows us to create an incident when a behavior-based detection is available. OAuth apps are not listed as “Impacted entities” and  the behaviors table cannot be used in Microsoft Sentinel (Microsoft 365 Defender Data Connector is not covering this table). Nevertheless,  custom detection seems is able to add the Entities to the incident.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon6.png)

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon7.png)

### Microsoft Defender 365 App Governance

App Governance offers various threat detection alerts for Service Principals.
The add-on feature for Microsoft Defender for Cloud Apps (MDA) is available without any additional costs for every customer with a valid license. Details about the licensing are described in the [Microsoft Learn article about App Governance Licensing](https://learn.microsoft.com/en-us/defender-cloud-apps/app-governance-get-started#licensing).

A list of the built-in threat detections including severity, description, TTP mapping and recommended detections are available in the [investigation guide for App Governance](https://learn.microsoft.com/en-us/defender-cloud-apps/app-governance-get-started#licensing).

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon8.png)

*In addition, templates but also custom policies can be configured to create alerts based on self-defined conditions and scopes. For example, detections for increased data usage for application without verified (ISV) publisher.*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon9.png)

*Alert will be generated with the evidence of the related OAuth application in Microsoft 365 Defender.* 

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon10.png)

*The alert will be forwarded to Microsoft Sentinel as incident (via M365D connector).*

Unfortunately, the entity mapping to the OAuth application is not included. Furthermore, there is no single value in the `SecurityAlert` event which can be used to build a correlation and mapping to the application. However, the description includes the `OAuthAppId` as part of the deep link to the OAuth app summary page in M365D Portal.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon11.png)

The following KQL query shows how we could extract the `OAuthAppId`from the URL:

```powershell
SecurityAlert
| where ProductName == "Microsoft Application Protection"
| extend CloudAppUrl = parse_url(Description)
| extend CloudAppUrlParam = parse_json(tostring(CloudAppUrl.["Query Parameters"])).oauthAppId
| extend AppId = tostring(toguid(CloudAppUrlParam))
| extend Category = tostring(parse_json(ExtendedProperties).Category)
| extend AlertDisplayName = tostring(DisplayName)
| distinct TimeGenerated, AppId, AlertSeverity, AlertDisplayName, Category
| project
    TimeGenerated,
    AppId,
    AlertSeverity,
    AlertDisplayName,
    Category
```

The result of the query is a list of all App Governance Alerts and the related `App Id`

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon12.png)

This query can be used to implement a analytics rule to create incidents in Microsoft Sentinel with the mapping of the AppId to the entity type “Cloud Application”:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon13.png)

### Entra ID Protection for Workload Identities

Microsoft has introduced “Identity Protection for Workload Identities” as part of Microsoft Entra Workload ID Premium. This offers a couple of [Risk detection](https://learn.microsoft.com/en-us/azure/active-directory/identity-protection/concept-workload-identity-risk#workload-identity-risk-detections) which will be considered to calculate the sign-in and risk of workload identities. One of my favorite capabilities are the behavior-based detections for “Suspicious sign-ins” and “Anomalous service principal activity”.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon14.png)

*The risk detections and details are available from the Entra ID Protection blade.*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon15.png)

*Alerts are also visible in the Microsoft 365 Defender portal but without any assets or evidence details.*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon16.png)

*Incidents will be created also in Microsoft Sentinel but without any mapping to the “Cloud Application” entity type.*
*Side note: A few alerts will be only forwarded when “All alerts” are configured (instead of “High-impact alerts only”) which needs to be configured in the [Alert service settings](https://learn.microsoft.com/en-us/microsoft-365/security/defender/investigate-alerts?view=o365-worldwide#configure-microsoft-entra-ip-alert-service).*

Details about the related Service Principal to the risk detection are available in the `SecurityAlert` event entry, as we can see in the following screenshot:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon17.png)

This allows the correlation to other alerts based on the `AppId`. For example, enrichment of Risk Detections with App Governance alerts of an application.

```powershell
AADServicePrincipalRiskEvents
| join kind=innerunique (SecurityAlert
| where ProductName == "Microsoft Application Protection"
| extend CloudAppUrl = parse_url(Description)
| extend CloudAppUrlParam = parse_json(tostring(CloudAppUrl.["Query Parameters"])).oauthAppId
| extend AppId = tostring(toguid(CloudAppUrlParam))
| extend Category = tostring(parse_json(ExtendedProperties).Category)
| extend AlertDisplayName = tostring(DisplayName)
| distinct AppId, AlertSeverity, AlertDisplayName, Category)
on $left.AppId == $right.AppId
| project AppId, ServicePrincipalId, ServicePrincipalDisplayName, RiskLevel, AlertSeverity, AlertDisplayName, Category
```

This query can be used to build an analytic rule with mapping to the “Cloud Application” entity type.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon18.png)

*Incident entity mapping by using `AppId` for “Cloud Application” entity type.*

### Microsoft Sentinel

**Analytic Rule Templates**

Microsoft has released a [security operations guide](https://learn.microsoft.com/en-us/entra/architecture/security-operations-applications) for Microsoft Entra which also covers  recommendations for applications. This includes what type of activities and events should be monitored and where to find them. Rule templates for Microsoft Sentinel are available for the most named detection use cases and can be found as a link in the document. Nevertheless, you should verify the logic behind the queries and consider customizing them to your environment and requirements. Enrichment can help to reduce noise or implement dynamic severity of incidents based on environment-specific conditions. In the next part of this blog post series, we will have a look on the use case “Changes to application ownership” in combination with enrichment of the included entities.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon19.png)

*Content Hub Solution “Azure Active Directory” offers many rule templates for Application security monitoring. Keep an eye on the entity mapping of analytics rules. They should include  the entity type “Application” for building correlation to other incidents. As we can see in the “Similar incidents” area of the incident page, correlation to other incidents will be established and increased visibility for multi-stage attacks.*

**Anomalies and Behavior Analytics**

User Entity Behavior Analytics (UEBA) is analyzing the behavior of users over a period of time and constructing a baseline of legitimate activity. This feature is integrated to Microsoft Sentinel and can be easily enabled. The `BehaviorAnalytics` can be used for customized or advanced anomaly detections of Application Management.

The following KQL query can be used to get all Audit events in the category “Application Management” with Insights from UEBA:

```jsx
BehaviorAnalytics
| where ActivityType == "ApplicationManagement"
// Optional: Filter events with Investigation Priority Score
//| where InvestigationPriority != 0
| project TimeGenerated, ActivityType, ActionType, ActivityInsights, UserPrincipalName, SourceIPAddress, SourceIPLocation, InvestigationPriority
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon20.png)

*UEBA includes an Investigation Priority Score and insights about the activity. In this case, it is a first-time operation from this user’s location.*

The `Anomalies` table allows us to get specific anomaly detection with details about the activity:

```jsx
Anomalies
| where RuleId == "a255ca7d-ea19-4b7b-8d88-a51ce1c72c29"
| project AnomalyTemplateName, RuleName, Description, Tactics, Techniques, AnomalyDetails, AnomalyReasons
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon21.png)

*Detailed anomaly insights of a user who performed an app role assignment for the very first time.*

**Detecting multi attack activities by Microsoft Sentinel Fusion**

Microsoft Sentinel Fusion is also correlating [suspicious sign-in events (detected by Entra ID protection) which leads to rare application consent](https://learn.microsoft.com/en-us/azure/sentinel/fusion-scenario-reference#rare-application-consent-following-suspicious-sign-in) (detected by the associated scheduled analytics rules). A multi-stage attack incident will also be created if entities mapped in combination of this kind of anomalies.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-03-entra-workload-id-threat-detection/workloadidsecmon22.png)

*Various incidents from analytics rules with entity mapping of the same IP address and user names matches with an anomaly detection  of Azure operations*

## Next: Advanced Detections and Enrichment

In the next part of this blog post series, we will go into details how we can improve the entity mapping and context of the incidents in Microsoft Sentinel by using enrichment and custom analytics rules. This includes also how to identify high-privileged workload identities and many examples for attack scenarios and related detection capabilities.