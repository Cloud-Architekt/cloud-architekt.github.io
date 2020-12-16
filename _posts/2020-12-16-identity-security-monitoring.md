---
layout: post
title:  "Identity Security Monitoring in Microsoft Cloud Services"
author: thomas
categories: [ Azure, Security, AzureAD ]
tags: [security, azuread, azure]
image: assets/images/azsentinel.png
description: "Microsoft offers several solutions and services for securing (hybrid) identities and protecting access to workloads such as Azure, Office 365 or other integrated apps in Azure Active Directory. I like to give an overview about data sources or signals that should be considered for monitoring based on identity-related activities, risk detections, alerts and events across the Microsoft ecosystem"
featured: false
hidden: false
---

*Microsoft offers several solutions and services for securing (hybrid) identities and protecting access to workloads such as Azure, Office 365 or other integrated apps in Azure Active Directory. I like to give a detailed overview about data sources or signals that should be considered for monitoring based on identity-related activities, risk detections, alerts and events across the Microsoft ecosystem.*

#### Table of Content:
- <A href="#identity-security-monitoring-in-a-hybrid-environment">Identity Security Monitoring in a "Hybrid Environment"</A><br>
- <A href="#azure-monitor-operational-logs-and-alerts-of-azure-ad-and-azure-workloads">Azure Monitor:<br>
Operational Logs and Alerts of "Azure AD" and "Azure Workloads"</A><br>
- <A href="#mcas-and-defender-for-identity-unified-secops-of-connected-cloud-apps-and-hybrid-identity">Microsoft Cloud App Security and Defender for Identity:<br>
Unified SecOps of connected "Cloud Apps" and "Hybrid Identity"</A><br>
- <A href="#microsoft-365-defender-unified-secops-of-m365-services">Microsoft 365 Defender:<br>
Unified SecOps of M365 Services</A><br>
- <A href="#azure-sentinel-single-pane-of-glass-across-azure-microsoft-365-and-3rd-party-cloud-platforms">Azure Sentinel:<br>
“Single pane of glass” across Azure, Microsoft 365 and "3rd party solutions"</A>

## Identity Security Monitoring in a Hybrid Environment

In the recent year, I‘ve talked about monitoring of Azure Active Directory in community sessions and talks. From my point of view a comprehensive monitoring of "identity security events must be an essential part of deployment plans and daily operations. Even though it may seem obvious as „best practice“ („[Actively monitor for suspicious activities](https://docs.microsoft.com/en-us/azure/security/fundamentals/identity-management-best-practices#actively-monitor-for-suspicious-activities)“), it is sometimes underrated.

Monitoring across "Azure AD" and "Active Directory" (including spreading between workloads in Azure and on-premises environments) can be complex and sometimes challenging but more important then ever. Identity protection in a "hybrid world" means also to protect and monitor Active Directory environments with all existing risks and traditional attack methods (e.g. "Pass-the-Hash"). The weakest link and uncover attack surfaces in your on-premises environment can be used to leverage or extend attacks to (hybrid) cloud services.

![../2020-12-16-identity-security-monitoring/AzIdentity_Security.png](../2020-12-16-identity-security-monitoring/AzIdentity_Security.png)
_"Microsoft Defender for Identity" (MDI), "Microsoft Cloud App Security" (MCAS) and "Azure AD Identity Protection" protects identities on various levels and platforms (On-Premises, Session/Cloud Apps and Cloud Identity/Sign-ins)_ 

Implementing "identity security" does not end with „enabling“ those features or by following the recommendations by „[Identity Secure Score](https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/identity-secure-score)“...

It’s important to develop a "continuous improvement" strategy of detections and "operational guide" to empower and monitor your signals of „guards“. This includes also to provide workflows for automated response, an "unified view" for incident management/hunting, security processes and posture management.

Extensive possibilities of "User and Entity Behavior Analytics" (UEBA) allows SecOps to find anomalous activities (calculated by machine learning algorithms) across the various data sources or signals instead of building a manual correlation.

At always, keep up-to-date and notified about latest changes of security features, attack/defense scenarios or security recommendations. Verification of effectiveness by "simulated attacks" should be also part of your operational tasks.

This blog post is an attempt to give a detailed overview on solutions to collect identity-related security events and implement auto-response on threads or risks.
Many links to detailed documentation by Microsoft and members of the community are included.
There is no claim for completeness and comprehensive view of all options.
I've tried to find some "sample use cases" to underline when this monitoring option will be in particularly relevance.
It was hard for me to find the right level of details or scope with regard to the wide-range of this topic.

*The following objectives are excluded and out of scope:
Azure AD B2C, Azure AD Domain Services and Microsoft Information Protection (AIP/MIP) will not be described in this blog post.*

_Caution: All description of features, potential limitations and implementation considerations are based on personal experiences and research results at the time of writting this blog post. Therefore the content and statement can be outdated since the article was published in December 2020._

*Note: Microsoft announced many product name changes at the Ignite 2020. I've used all new product names in this article.
A good overview of all name changes are included [in this blog post by Microsoft](https://techcommunity.microsoft.com/t5/itops-talk-blog/microsoft-365-and-azure-security-product-name-changes/ba-p/1719167).*

## Azure Monitor: Operational Logs and Alerts of Azure AD and Azure Workloads

*Sample use case: Security Operation Teams (SecOps) manages Microsoft Azure workloads only (no M365 services) and needs an "unified view" of Azure Services and Azure AD security events. No hybrid identity (Windows Server Active Directory) or hybrid cloud (Google Cloud, AWS) scenarios.*

![../2020-12-16-identity-security-monitoring/AzIdentity_AzMonitor.png](../2020-12-16-identity-security-monitoring/AzIdentity_AzMonitor.png)

### Data Sources of "Azure Monitor Logs"

#### IaaS/PaaS (Cloud and on-Premises) in "Azure Monitor"

*[Azure Monitor](https://docs.microsoft.com/en-us/azure/azure-monitor/) allows you to collect logs from the Azure platform and resources for visualization and alerting or forwarding to other destination (for long-term retention or advanced scenarios). In this use case we are using Microsoft "Log Analytics" to enable advanced (KQL-based) queries and centralized collection of logs*. *The following data sources should be considered to collect relevant information for your IAM security monitoring:*

- [Azure Activity Logs](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/view-activity-logs):
Platform logs of Azure which includes details on various events in your subscriptions and resources (including administrative activities, service health, recommendations).
Changes of Azure RBAC are also part of this activity logs.
    - [Collect and analyze Azure Activity Logs in Azure Monitor](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/activity-log-collect)
    - [Data schema of Azure Audit Logs](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/activity-log-schema)
- [Azure Resource Logs (Diagnostic Logs)](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/resource-logs):
Insights and "Audit Logs" of operations that were performed within an "Azure resource" are stored here. This includes the logging of identity and access (IAM)-related services such as "Azure KeyVault". You need to configure the [diagnostic settings to monitor access](https://docs.microsoft.com/en-us/azure/key-vault/general/howto-logging) of secrets and certificates which are stored in the vault.
- [Storage of Security Events in Log Analytics](https://docs.microsoft.com/en-us/azure/security-center/security-center-enable-data-collection#data-collection-tier):
Collect all (security) events from servers in Azure and non-Azure/On-Premises infrastructure as part of the Azure Security Center and Data Collection.
    - [Collect data from physical/virtual server (hybrid environments) with Azure Monitor](https://docs.microsoft.com/en-us/azure/azure-monitor/learn/quick-collect-windows-computer) and Log Analytics Agent (Event Logs and Performance Counter)
    - Data Collection of "security logs" to Log Analytics can be [configured in "Azure Security Center"](https://docs.microsoft.com/en-us/azure/security-center/security-center-enable-data-collection#what-event-types-are-stored-for-common-and-minimal) by choosing "All Events", "Minimal" or "Common" options. Therefore it's strongly recommended to use the same workspace for "Azure Security Center" and "Azure Monitor" logs.
    - Use cases for (Active Directory) Domain Controller log queries:
        - [Query Active Directory Security Events using Azure Log Analytics and Azure Security Center (Blog post by Richard Hooper / Pixel Robots)](https://pixelrobots.co.uk/2019/07/query-active-directory-security-events-using-azure-log-analytics-on-the-cheap/)
        - [Using Azure Security Center and Log Analytics to Audit Use of NTLM](https://techcommunity.microsoft.com/t5/core-infrastructure-and-security/using-azure-security-center-and-log-analytics-to-audit-use-of/ba-p/1077045)

#### Azure Security Center (ASC) and "Azure Monitor"

*[Continous Export](https://docs.microsoft.com/en-us/azure/security-center/continuous-export) allows to forward alerts and recommendations to "Azure Event Hub" or "Log Analytics". This solution is divided in two different scopes: Free service "Security Center" as "Cloud security posture management (CSPM)" solution. Azure Defender as "Cloud workload protection (CWP)" add-on with licensing option to pay only for what you use.*

- [Recommendations](https://docs.microsoft.com/en-us/azure/security-center/recommendations-reference#recs-identity):
Checks on-boarded subscriptions and their resources of recommendations around [Identity and access „resource security“](https://docs.microsoft.com/en-us/azure/security-center/security-center-identity-access).
    - [Reference Guide on Identity & Access Recommendations](https://docs.microsoft.com/en-us/azure/security-center/recommendations-reference#recs-identity)
- [Alerts](https://docs.microsoft.com/en-us/azure/security-center/alerts-reference):
Various types of IaaS and PaaS resources (VMs, App Service, Storage,…) will be protected against threats including identity-related attacks (e.g. „A logon from a malicious IP has been detected“) or malware (e.g. Mimikatz or any "attack tools"). Triggering of alerts can be tested as described in the „[Alert validation](https://docs.microsoft.com/en-us/azure/security-center/security-center-alert-validation#validate-alerts-on-windows-vms-)“ guide of Microsoft.
    - [Azure Defender for Servers](https://docs.microsoft.com/en-us/azure/security-center/defender-for-servers-introduction) and [Integration of Microsoft Defender for Endpoint](https://docs.microsoft.com/en-us/azure/security-center/security-center-wdatp):
    Alerts for [Windows](https://docs.microsoft.com/en-us/azure/security-center/alerts-reference#alerts-windows) and [Linux](https://docs.microsoft.com/en-us/azure/security-center/alerts-reference#alerts-linux) covers attack sources such as "anonymous or malicious IP addresses" and integrates "Microsoft Defender for Endpoint" for extended scenarios.
    You‘ll get advanced post-breach detections and alerts from "Endpoint Protection", alongside of automation of onboarding sensors.
        - [Playbook for Windows Servers](https://github.com/Azure/Azure-Security-Center/blob/master/Simulations/Azure%20Security%20Center%20Security%20Alerts%20Playbook_v2.pdf) includes step-by-step instruction to simulate attacks (such as "lateral movement").
        - [Audit logs of requests to "Just-in-Time Access"](https://docs.microsoft.com/en-us/azure/security-center/alerts-reference) are available in the "Activity Logs".
    - Azure Defender for PaaS and threat protection capabilities:
    Various Azure PaaS services such as [Azure App Service](https://docs.microsoft.com/en-us/azure/security-center/defender-for-app-service-introduction), [containers](https://docs.microsoft.com/en-us/azure/security-center/container-security), [SQL](https://docs.microsoft.com/en-us/azure/security-center/defender-for-sql-introduction) or [KeyVault](https://docs.microsoft.com/en-us/azure/security-center/defender-for-key-vault-introduction) can be also protected by "Azure Defender". Other platform services as the [management layer of Azure (Resource Manager)](https://docs.microsoft.com/en-us/azure/security-center/other-threat-protections) or [network layer](https://docs.microsoft.com/en-us/azure/security-center/other-threat-protections#threat-protection-for-azure-network-layer-) are also included. Detection of unusual or risky operations in the Azure subscription [are powered by MCAS](https://docs.microsoft.com/en-us/azure/security-center/alerts-reference#alerts-azureresourceman).
- [Data schema](https://docs.microsoft.com/en-us/azure/security-center/alerts-schemas?tabs=schema-continuousexport) describes fields and data types of the ASC security alerts.

### Cloud Identity (Azure Active Directory) in "Azure Monitor"

[Routing of Azure AD activity logs](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/concept-activity-logs-azure-monitor) is natively supported to various targets such as Azure Event Hub, Blob Storage and Log Analytics.

*Supported reports in Azure Monitor*

- [Azure AD Sign-In Logs](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/concept-sign-ins):
Overview of authentication and authorization events of all users.
Details on the content are defined in the [sign-in logs schema](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/reference-azure-monitor-sign-ins-log-schema)
- [Azure AD Audit Logs:](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/concept-audit-logs)
Activities of tasks that is performed by a user or admin of your Azure AD tenant.
[Audit Log schema](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/reference-azure-monitor-audit-log-schema) defines the content of this activity log.
- [Azure AD Non-Interactive Logs:](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/concept-all-sign-ins)
Microsoft updated the logging capabilities in Azure AD as addition to the above mentioned classic "Azure AD Reports".
    - ["Service Principal" sign-ins](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/aadserviceprincipalsigninlogs)
    - ["Non-interactive user" sign-ins](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/aadnoninteractiveusersigninlogs)
    - ["Managed identity" for Azure resource sign-ins](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/aadmanagedidentitysigninlogs)
- [Azure AD Provisioning Logs:](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/concept-provisioning-logs)
This log gives you [detailed insights of provisioning](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/concept-provisioning-logs) users, roles and groups from or to Azure AD. [Log schema for Azure Monitor](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/aadprovisioninglogs) is also documented in MSDocs.

*Non-supported reports in Azure Monitor*

- Azure AD (Identity Protection) Security Logs:
Identity Protection of Azure AD Premium [stores reports and events](https://docs.microsoft.com/en-us/azure/active-directory/identity-protection/howto-identity-protection-investigate-risk) of risky users, sign-ins (up to 30 days) and detections (up to 90 days).
    - Various "[risk detections](https://docs.microsoft.com/en-us/azure/active-directory/identity-protection/concept-identity-protection-risks)" are available which will be calculated in real-time or offline.
    Risk state triggers "auto-response actions" and offers "self-remediation options" that will be managed by [identity protection policies](https://docs.microsoft.com/en-us/azure/active-directory/identity-protection/concept-identity-protection-policies) or as part of Conditional Access Policies.
    - [Microsoft Graph APIs](https://docs.microsoft.com/en-us/azure/active-directory/identity-protection/howto-identity-protection-graph-api) allows you to collect this data for export or automate response to risk detections.
    - [Every Azure AD Sign-in log](https://docs.microsoft.com/en-us/graph/api/signin-list?view=graph-rest-1.0&tabs=http) includes the following properties related to the identity risk detection: riskDetail, riskLevelAggregated, riskLevelDuringSignIn, riskState, riskEventTypes.

        ![../2020-12-16-identity-security-monitoring/AzIdentity_AzMonitor_IPCDetection.png](../2020-12-16-identity-security-monitoring/AzIdentity_AzMonitor_IPCDetection.png)
        _Risk detections from "Cloud App Security" (such as "Impossible Travel") will be also displayed in the "Identity Protection" blade (Azure portal). Correlation between sign-in event and offline detections by Identity Protection (in this sample "Password Spray, Malicious IP address and Atypical travel) can be established by Request or CorrelationID._

        ![../2020-12-16-identity-security-monitoring/AzIdentity_AzMonitor_IPCQuery.png](../2020-12-16-identity-security-monitoring/AzIdentity_AzMonitor_IPCQuery.png)
        _Collected "sign-in events" in "Azure Monitor Logs" will be enriched with "RiskState" and "RiskLevelDuringSign" if a risky sign-in was detected (in real-time or sign-in attempt was made after risk detection)._

### Log Collection (via "Azure Monitor")

- [Log Analytics](https://docs.microsoft.com/en-us/azure/azure-monitor/log-query/log-query-overview):
Azure's native "Log Management Solution" enables deep analytics and advanced queries in case of troubleshooting or technical investigation. [Kusto (KQL)](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/) is the query language that you should use (and learn).
This is the foundation of many features and primary query language in solutions such as as "Azure Sentinel" (built on top of Log Analytics) or "Azure AD Workbooks" (sourced log data from a workspace). Other monitoring solutions in the "Azure platform" are using also "Log Analytics workspaces" to store data (e.g. App Insights).

### Analyze and Visualize with "Azure Monitor"

- [Azure AD Workbooks](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/howto-use-azure-monitor-workbooks):
Microsoft provides built-in visualization which requires Log Analytics workspace. They are very helpful to analyze (and troubleshoot) activities around Authentication, Conditional Access and other identity-related operations.
- [Azure AD Dashboards](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/howto-install-use-log-analytics-views):
Azure AD Dashboard views are available for "Account Provisioning" and "Sign-In Events" but seems little bit outdated.
- [Usage and Insights](https://docs.microsoft.com/en-us/azure/active-directory/authentication/howto-authentication-methods-usage-insights):
An overview about registered/usage of "authentication methods" and "Azure AD-integrated apps" activity are available in the "Azure Portal". These usage statistics are also available via "[Microsoft Graph API](https://docs.microsoft.com/en-us/graph/api/resources/authenticationmethods-usage-insights-overview?view=graph-rest-beta)". The following [sample script](https://docs.microsoft.com/en-us/samples/azure-samples/azure-mfa-authentication-method-analysis/azure-mfa-authentication-method-analysis/) analyse the statistics to make recommendations.
- Log Analytics Solutions:
    - [AD Health Check](https://docs.microsoft.com/en-us/azure/azure-monitor/insights/ad-assessment):
    Optimize your Active Directory environment with the "Active Directory Health Check" solution in Azure Monitor
    - [AD Replication Status](https://docs.microsoft.com/en-us/azure/azure-monitor/insights/ad-replication-status):
    Monitor your "Active Directory replication status" with Azure Monitor

### Integration and Response in "Azure Monitor"

- [Alerts and Logic Apps](https://docs.microsoft.com/en-us/azure/active-directory/authentication/howto-authentication-methods-usage-insights):
Azure Monitor is able to trigger complex actions based on defined rules (such as [signal logic based on KQL custom log search](https://docs.microsoft.com/en-us/azure/active-directory/authentication/howto-authentication-methods-usage-insights)). This gives you many options to integrate your workflows as part of "Azure Logic Apps" or any other 3rd Party systems.
    - Samples of "Azure Monitor Alerts":
        - [Monitor your Azure AD Break Glass Accounts with Azure Monitor
        (Blog post by Daniel Chronlund](https://danielchronlund.com/2020/01/22/monitor-your-azure-ad-break-glass-accounts-with-azure-monitor/))
        - [Azure AD App Tracking with Logic Apps (Blog post by Microsoft Developer Support)](https://devblogs.microsoft.com/premier-developer/azure-ad-app-tracking-with-logic-apps/)

### Considerations and References of Azure AD Logging by "Azure Monitor"

- Microsoft’s “[deployment guide of Azure AD Monitoring](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/plan-monitoring-and-reporting)” gives you an general overview of aspects and options to integrate or archive logs. The [latency of Azure AD logging and the risk detections](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/reference-reports-data-retention) should be also considered (for your security response and processes).
- [Retention of the reports](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/reference-reports-data-retention) depends on type of activity and your Azure AD license.
- [Costs](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/concept-activity-logs-azure-monitor#azure-monitor-logs-cost-considerations) should be calculated based on the requirements for long-term retention.
- Pay attention to missing audit logs of privileged activities in Azure Monitor.
    - Example: [Azure EA portal and changes of ownership will not be audited](https://www.cloud-architekt.net/azure-ea-management-security-considerations/) but has effects on Azure RBAC!
- [Rate limitation of „Azure Alerts“](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/alerts-rate-limiting) should be considered for your service emergency and operational notifications.
- [Identity Protection provides new APIs](https://docs.microsoft.com/en-us/graph/api/resources/identityprotection-root?view=graph-rest-beta) to get events and risk status of your users.
- I can strongly recommended to test and validate the detection mechanism in "Identity Protection" (as described in the [blog post by Sami Lampuu](https://samilamppu.com/2020/10/09/azure-ad-identity-protection-deep-diver-part-2/)). Take care on the delay and latency between attack and detection of the various mechanism. Keep in mind, risk response (as part of the "Risk Policies") must be in accord with your Azure AD implementation.
    - Example: Force password change by risky sign-in detection works only in hybrid environments if password write-back is allowed.
- Consider the limitations and behavior of "[Identity Protection and External Users (B2B Guests)](https://docs.microsoft.com/en-us/azure/active-directory/identity-protection/concept-identity-protection-b2b)"
- Hybrid identity environment: Collect and monitor [logs from "Azure AD Connect" servers](https://docs.microsoft.com/en-gb/troubleshoot/azure/active-directory/installation-configuration-wizard-errors#troubleshoot-additional-error-messages) and the ["Password Hash Sync" agent](https://docs.microsoft.com/en-gb/troubleshoot/azure/active-directory/troubleshoot-pwd-sync#event-id-messages-in-event-viewer).
- You might be facing the follow considerations in your daily work experiences:
    - Name in reports are based on the object name at the time of the event/sign-in
    - B2B users are able to get “user insights” and therefore internal information
    - Users can review the sign-in history as part of the "[My-Sign-Ins](https://techcommunity.microsoft.com/t5/azure-active-directory-identity/users-can-now-check-their-sign-in-history-for-unusual-activity/ba-p/916066)" portal and give feedback on suspicious activities ("This wasn't me" option).
    - Non-Global Admins can access logs
        - Security Administrator & Reader
        - Reports Reader and Application Administrator
        - Global Reader
- Microsoft "[Identity Secure Score](https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/identity-secure-score)" is recommended for regular check as part of your (cloud) "identity security posture management" and can be integrated in your monitoring via [Security Graph API](https://docs.microsoft.com/en-us/graph/api/securescore-get?view=graph-rest-1.0&tabs=http).
- [Customer-managed keys](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/customer-managed-keys) can be configured and are supported for encryption in "Azure Monitor".
- Self-Service Password Reset (SSPR): Monitor blocked attempts or suspicious activity via [Azure Monitor Alerts or Azure Sentinel](https://www.cloud-architekt.net/azuread-sspr-deployment-and-detection/)
- Consider service principal logs and the challenges to build relation to "Azure Activity" or "(Azure) DevOps Deployment pipeline" logs, [as described in my previous blog post](https://www.cloud-architekt.net/auditing-of-msi-and-service-principals/)!

## MCAS and "Defender for Identity": Unified SecOps of connected "Cloud Apps" and "Hybrid Identity"

![../2020-12-16-identity-security-monitoring/AzIdentity_MCAS.png](../2020-12-16-identity-security-monitoring/AzIdentity_MCAS.png)

*Sample use case: SecOps that manages security of cloud platforms or SaaS solutions and need an unified view for investigation or alerting on (hybrid) identities.*

### Data Sources in MCAS

#### IaaS/PaaS (Cloud and on-Premises) in MCAS

*MCAS allows you to [connect Microsoft‘s Azure platform](https://docs.microsoft.com/en-us/cloud-app-security/connect-azure-to-microsoft-cloud-app-security) and other cloud platform provider ([AWS](https://docs.microsoft.com/en-us/cloud-app-security/connect-aws-to-microsoft-cloud-app-security) and [Google Cloud Platform](https://docs.microsoft.com/en-us/cloud-app-security/connect-google-gcp-to-microsoft-cloud-app-security)) via "App Connector". This makes the „Activity logs“ available in MCAS for investigation and trigger alerts. The security configuration of "Google Cloud" and "Amazon Web Services" (AWS) can be integrated to provide fundamental security recommendations based on the CIS benchmark.* 

#### Azure Security Center (ASC) and MCAS-Integration

*Security Configuration [Assessment results](https://docs.microsoft.com/en-us/cloud-app-security/security-config) of MCAS will be collected from the "Azure Security Center". This gives you a common view of the [security posture, usage of cloud resources and suspicious activities](https://docs.microsoft.com/en-us/cloud-app-security/tutorial-cloud-platform-security) across your cloud infrastructure assets (in Microsoft Azure).*

#### Cloud Identity in MCAS

*Azure AD audit and sign-in events are covered by the „Office 365 connector“ in MCAS. But only interactive sign-in activities and sign-in activities from legacy protocols seems to be included.
Advanced categories such as "Service Principals" aren't covered by the connector (yet).* 

- Severity of (cloud) identity risk alerts can be managed by [MCAS integration of "AAD Identity Protection"](https://docs.microsoft.com/en-us/cloud-app-security/aadip-integration).
- Detections of "Identity Protection" are part of the UEBA/investigation priority score and will be also displayed on the user page.
    - Some identity [risk detections will be detected by MCAS](https://docs.microsoft.com/en-us/azure/active-directory/identity-protection/concept-identity-protection-risks#sign-in-risk) (such as "impossible travel" or "inbox manipulation rule") and results will be forwarded to Azure AD Identity Protection (and displayed in the portal blade as well).
    Follow the [Identity protection playbook](https://docs.microsoft.com/en-us/azure/active-directory/identity-protection/howto-identity-protection-simulate-risk) to simulate the attack scenarios and trigger the detection/response.

![../2020-12-16-identity-security-monitoring/AzIdentity_MCAS_IPCAlerts.png](../2020-12-16-identity-security-monitoring/AzIdentity_MCAS_IPCAlerts.png)
_All "Identity Protection" risk detections will be listed in the MCAS alerts view._

#### On-Premises Identity (Active Directory) in MCAS

[Microsoft Defender for Identity](https://docs.microsoft.com/en-us/azure-advanced-threat-protection/what-is-atp) (MDI) allows you to detect and identify attacks in your "on-premises environment". It‘s based on monitoring and profiling of user behavior and activities. MDI includes the [detection of Lateral Movement Paths (LMP)](https://techcommunity.microsoft.com/t5/microsoft-security-and/reduce-your-potential-attack-surface-using-azure-atp-lateral/ba-p/291787) which is strongly recommended to consider. Keep in mind, machine-learning on user/entity-related behavioral needs a learning period. 

- [Security Alerts](https://docs.microsoft.com/en-us/azure-advanced-threat-protection/suspicious-activity-guide?tabs=cloud-app-security#security-alert-name-mapping-and-unique-external-ids) are categorized in phases of the "CyberAttack kill-chain" and are [well documented](https://docs.microsoft.com/en-us/defender-for-identity/suspicious-activity-guide?tabs=external) by Microsoft.
- [Integration of MDI with Defender for Endpoint](https://docs.microsoft.com/en-us/defender-for-identity/integrate-mde#:~:text=In%20the%20Defender%20for%20Identity,the%20integration%20toggle%20to%20On.&text=To%20check%20the%20status%20of,Microsoft%20Defender%20for%20Endpoint%20integration.) allows you to laverage the detection from events of the domain controller to all endpoints. Furthermore it allows you to see [open MDI alerts as badge in the MDE portal](https://docs.microsoft.com/en-us/defender-for-identity/investigate-entity#cross-check-with-windows-defender).
- [Integration of MDI with MCAS](https://docs.microsoft.com/en-us/defender-for-identity/mcas-integration) is mandatory for environments where both solutions are in use. More and more features around MDI seems to be moved to the MCAS portal (such as the new "hunting experiences"). All activities and detections of MDI will be also included in the UEBA if integration is configured.


    ![../2020-12-16-identity-security-monitoring/AzIdentity_MCAS_MDIPortal.png](../2020-12-16-identity-security-monitoring/AzIdentity_MCAS_MDIPortal.png)
    _Attacks on Active Directory (On-Premises) will be detected by MDI and generates an alert in the MDI portal._

    ![../2020-12-16-identity-security-monitoring/AzIdentity_MCAS_MDIAlertInMCAS.png](../2020-12-16-identity-security-monitoring/AzIdentity_MCAS_MDIAlertInMCAS.png)
    _New "hunting experience" allows to use MCAS portal for unified incident management between "Active Directory" and "Azure AD" alerts._ 

    - MCAS is using MDI data as source to collect activities from Active Directory as an „app“. This gives you an "unified activity" overview of an user in "Azure AD", "Active Directory" and "MCAS connected apps".


    ![../2020-12-16-identity-security-monitoring/AzIdentity_MCAS_UnifiedFailedLogins.png](../2020-12-16-identity-security-monitoring/AzIdentity_MCAS_UnifiedFailedLogins.png)
    _Example: Failed sign-in attempts to Active Directory or connected apps (in this case, "Azure Portal" and "Office 365")._

- All alerts and events of MDI can be exported in [CEF format](https://docs.microsoft.com/en-us/defender-for-identity/cef-format-sa).
- Regular updates are adding new detection methods that raised up. A good example is the [vulnerability detection of ZeroLogon](https://www.microsoft.com/security/blog/2020/11/30/zerologon-is-now-detected-by-microsoft-defender-for-identity/) which was introduced in November 2020.
- Details on managing and investigation are very well documented in a TechCommunity blog article about ["daily operation of MDI"](https://techcommunity.microsoft.com/t5/core-infrastructure-and-security/microsoft-defender-for-identity-azure-atp-daily-operation/ba-p/1831024). Another article in this series describes details on ["Deployment and Troubleshooting"](https://techcommunity.microsoft.com/t5/core-infrastructure-and-security/microsoft-defender-for-identity-azure-atp-deployment-and/ba-p/1676122).

#### Cloud Session Monitoring by Cloud App Security

*Microsoft Cloud Access Security Broker (CASB) enables you to identify usage of cloud apps, insights of risk assessments and capabilities to control the sessions and access to the cloud apps. There are some features that are essential for monitoring your identities and their access to sanctioned or unsanctioned resources or apps:*

- "Cloud Discovery" allows you to detect usage of cloud apps from proxy/firewall logs or [machine-based discovery (from MDE)](https://docs.microsoft.com/en-us/cloud-app-security/mde-integration). Integrated "Cloud App Catalog" shows a "risk score" and detailed information (security factors, industry- and legal regulations) to discovered apps. Classification as "unsanctioned" will start blocking the usage of the app.
    - [Microsoft Defender for Endpoint (MDE) in MCAS](https://techcommunity.microsoft.com/t5/microsoft-security-and/microsoft-cloud-app-security-and-windows-defender-atp-better/ba-p/263265) has many benefits around ["Cloud Discovery"](https://docs.microsoft.com/en-us/cloud-app-security/mde-integration) (ingestion of discovered machine data) and the usage of [custom network indicators](https://techcommunity.microsoft.com/t5/microsoft-security-and/block-access-to-unsanctioned-apps-with-microsoft-defender-atp/ba-p/1121179) (to block unsanctioned cloud apps).
    - Discovery logs can be [enriched with "Azure AD" username data](https://docs.microsoft.com/en-us/cloud-app-security/cloud-discovery-aad-enrichment). This replaces the original username from the proxy or firewall traffic logs by the Azure AD user name.
- MCAS provides "[App Connectors](https://docs.microsoft.com/en-us/cloud-app-security/enable-instant-visibility-protection-and-governance-actions-for-your-apps)" for some cloud provider to get advanced visibility and protection of sanctioned/connected apps. Insights of user/access management, activity logs and file/sharing control will be collected via "API connections" to the supported cloud products (GitHub Enterprise, ServiceNow, etc.).
- Sessions to Azure AD-integrated apps can be managed (for limited access) in MCAS with "Conditional Access App Control". MCAS acts as a [reverse proxy to control sessions](https://docs.microsoft.com/en-us/cloud-app-security/proxy-intro-aad#how-it-works) of the configured cloud app. This gives you [various compliance options](https://docs.microsoft.com/en-us/cloud-app-security/tutorial-proxy) such as preventing data exfiltration or real-time monitoring of user activity (anomalies) in the session.

![../2020-12-16-identity-security-monitoring/AzIdentity_MCAS_CloudSessionAlerts.png](../2020-12-16-identity-security-monitoring/AzIdentity_MCAS_CloudSessionAlerts.png)
_MCAS allows to get insights of suspicious user behavior in the session to a connected cloud app (such as download/upload to OneDrive and SharePoint). Custom activity alerts are also possible (like in this example, activity by "Global Admin to gain elevated access to Azure Management scope")._

#### Collaboration Platforms (Office 365 Services) in MCAS

*Microsoft‘s Office 365 but also other collaboration platforms ([Dropbox](https://docs.microsoft.com/en-us/cloud-app-security/connect-dropbox-to-microsoft-cloud-app-security) and [G-Suite](https://docs.microsoft.com/en-us/cloud-app-security/connect-google-apps-to-microsoft-cloud-app-security)) or SaaS provider can be connected via "app connectors". This gives you options to visibility, governance and protection of those services. Level of centralized management in MCAS depends on abilities that are [supported by the connector](https://docs.microsoft.com/en-us/cloud-app-security/enable-instant-visibility-protection-and-governance-actions-for-your-apps).* 

- [App Connector of "Office 365"](https://docs.microsoft.com/en-us/cloud-app-security/connect-office-365-to-microsoft-cloud-app-security) will be also used to source the following "Azure AD" information and are prerequisites for the identity monitoring capabilities in MCAS:
    - Azure AD Users and groups (Meta information of those objects in the tenant)
    - Azure AD Management events (Audit Logs)
    - Sign-in events (Interactive Sign-in activities and activities from legacy protocols such as ActiveSync)
    - Azure AD Apps (registered "OAuth" apps)
- Audit data will be ingested from the "Office 365 Management Activity API". A deep dive description of collecting this audit events are very well described in [a blog post by Sami Lamppu](https://samilamppu.com/2020/08/07/office-365-audit-events-visibility-in-cloud-app-security/).
In general, the audit log from "Office 365 Security and Compliance Portal" shows the same level of information as the activity log from the MCAS "app connector".
    - Description of [audited activities in Office 365](https://docs.microsoft.com/en-us/microsoft-365/compliance/search-the-audit-log-in-security-and-compliance?view=o365-worldwide#audited-activities)
    - Detailed [properties of the Office 365 audit log](https://docs.microsoft.com/en-us/microsoft-365/compliance/detailed-properties-in-the-office-365-audit-log?view=o365-worldwide)
- Alerts of "Microsoft Defender for Office 365" (MDO) are also visible in the "MCAS Activity Log" and allows you to create custom policies in MCAS. Sami Lamppu has also given some details about this [in one of his blog posts](https://www.notion.so/ttps-samilamppu-com-2020-05-19-detect-o365-atp-alerts-in-cloud-app-security-and-microsoft-threat-p-399da36e4bf745e5861ef420873b1ad2).

#### Device / Endpoint Security (Microsoft Defender for Endpoint) Integration in MCAS

*"Microsoft Defender ATP" (MDATP) Portal was rebranded to "[Microsoft Defender Security Portal](https://docs.microsoft.com/en-us/windows/security/threat-protection/microsoft-defender-atp/portal-overview)" in the fall of 2020. Microsoft's Endpoint and Detection and Response (EDR) solution allows [deep integration to MCAS](https://techcommunity.microsoft.com/t5/microsoft-security-and/microsoft-cloud-app-security-and-windows-defender-atp-better/ba-p/263265). But also insights from MCAS will be displayed in the (new) Endpoint Security Portal.*

Signal forwarding from "Microsoft Defender for Endpoint" (MDE) to MCAS is an essential configuration to establish the visibility of "cloud app usage" and control of unwanted apps (as described previously). But it's also extend the "MCAS Discovery Dashboard" with an additional "machine-centric" view. This allows you to start investigation based on a specific machine and correlate statistics of risky apps, transaction/traffic and device risk (calculated by MDE) right from MCAS. A direct (deep) link to the machine object helps to continue the investigation in the "Microsoft Defender Security Center" (MDE Portal).

### Analyze and Visualize with MCAS

#### User and Entity Behavior Analytics (UEBA) in MCAS

- [(Hybrid) Identity Threat Investigation](https://www.youtube.com/watch?v=znsX3ssctNM): The "user page" in MCAS gives you an overview and correlated information from all connected apps or integrated "threat protection solutions" in a single view. This is also a landing page to get all related and collected information about activities (from the "Activity Log"), alerts (filtered by selected user) or actions by policies ("Governance Log").
User Exposure information shows summary from various sources (e.g. number of accounts that are correlated by app connector).
Activities of the users can be filtered and will be enriched by MCAS (such as Internal/External users, Status as admin account or critical role/group assignment).
    - User page and "hunting experiences" in "Microsoft Defender for Identity" will be probably moved to the MCAS portal.
    - Device page is also available for all MDI device objects and shows open alerts and summary of activities (logged on users, resource access, IPs)
- [Investigation Priority Score](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/concept-audit-logs):
This score helps to identify the riskiest users across the various signals, alerts and integrations. Therefore, the "Investigation Priority" pulls together signals from connected apps and integrated threat protections (MDI and "Azure AD Identity Protection") to find abnormal behavior and aggregate this into a single score value. This score will be displayed on the "MCAS dashboard" and in the user page for [further investigation](https://docs.microsoft.com/en-us/cloud-app-security/tutorial-ueba).
    - Matt Soseman has recorded a [YouTube video](https://mattsoseman.wordpress.com/2020/07/08/ueba-in-microsoft-cloud-app-security-user-entity-behavior-analytics/) about it which includes the calculation of the score and some demo on investigation in the "UEBA".
    - Consider to [tune the policies for anomaly detection](https://docs.microsoft.com/en-us/cloud-app-security/tutorial-suspicious-activity) and review the default governance actions after enabling the data sources (threat protections, connected apps and discovery logs).


    ![../2020-12-16-identity-security-monitoring/AzIdentity_MCAS_UEBA.png](../2020-12-16-identity-security-monitoring/AzIdentity_MCAS_UEBA.png)
    _Alerts and activities of the last 7 days will be shown in the user page only. "Investigation priority" only considered the threats within this time range. So keep in mind, the total number of "open alerts" in the "user threat" panel._

#### Identity Security Posture and Apps Inventory with MCAS

- [Identity Security Posture](https://docs.microsoft.com/en-us/defender-for-identity/isp-overview):
Assessment of various security critical "Active Directory" configurations that includes also instructions to remediate or resolve the findings.
- [Manage OAuth apps](https://docs.microsoft.com/en-us/cloud-app-security/manage-app-permissions):
Risky apps can be [investigated by the discovered "OAuth apps" in MCAS](https://docs.microsoft.com/en-us/cloud-app-security/investigate-risky-oauth). Furthermore, you should also be able to [configure an app policy](https://docs.microsoft.com/en-us/cloud-app-security/app-permission-policy) to monitor "OAuth apps" on various criteria (incl. permission level) and revoke the app (if needed).
    - Built-in ["OAuth" app anomaly detection policies](https://docs.microsoft.com/en-us/cloud-app-security/app-permission-policy#oauth-app-anomaly-detection-policies) are already configured to detect malicious consent grants, misleading publisher/product names or activity to suspicious apps.

#### Integration and Responds in MCAS

- [Policies:](https://docs.microsoft.com/en-us/cloud-app-security/control-cloud-apps-with-policies)
Control of your identities in connected apps/resources can be achieved by monitored activities and signals by MCAS. The following types of policies can be configured:
    - [Access Policies](https://docs.microsoft.com/en-us/cloud-app-security/access-policy-aad) gives you the ability for real-time monitoring and control the access on your "connected apps". The applied actions ("Monitor" or "Block") can be filtered on advanced criteria such as device tags, source of access (IP address), client app or user/app scope.
    - [Session Policies](https://docs.microsoft.com/en-us/cloud-app-security/session-policy-aad) enables real-time action and monitoring within the session and allows to block or restrict on specific activities (such as sharing or file control).
    - [Activity Policies](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/concept-audit-logs) enforces automated response on a specific or repeated activity to a connected app. As a response, governance actions can enforce security mechanism on API level by "connected app" (e.g. require sign-in prompt in Office 365), user account state in Azure AD (e.g. disable user) or custom playbook (PowerAutomate).
    Activities of tasks that is performed by a user or admin of your Azure AD tenant.
    - [Anomaly Detection Policies](https://docs.microsoft.com/en-us/cloud-app-security/anomaly-detection-policy) are enabled to find unusual activities and trigger an alert if unusual behavior was detected (different from user's regular activity). They are part of the UEBA and ML capabilities which are integrated in MCAS and displayed in the "User Page" / "Investigation Priority Score". Built-in policies covers activities from specific activities in a connected app (e.g. connected "Azure Instance" and "Multiple delete VM activities") to find anomalies of a single user session ("Unusual activities by user"). Some anomaly detection policies [can be tuned or scoped](https://docs.microsoft.com/en-us/cloud-app-security/anomaly-detection-policy#tune-anomaly-detection-policies) to adjust sensitivity or customize automated response.


    ![../2020-12-16-identity-security-monitoring/AzIdentity_MCAS_Governance.png](../2020-12-16-identity-security-monitoring/AzIdentity_MCAS_Governance.png)
    _Governance log shows actions (initiated by policies) of automated response on alerts (such as require user to sign-in again if a risky sign-in was detected)._

- [Policy Templates](https://docs.microsoft.com/en-us/cloud-app-security/policy-template-reference):
Before creating your own policies, check the built-in templates in MCAS that are ready for use. Many of them are essential and strongly recommended to be enabled and configured in your MCAS instance.
- [PowerAutomate](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/concept-audit-logs):
Automation of (governance) actions can be realized by "PowerAutomate". Microsoft has released some sample playbooks for auto-remediation or -response scenarios on [GitHub](https://github.com/microsoft/Microsoft-Cloud-App-Security/tree/master/Playbooks).
    - Other samples such as "[Request user validation](https://techcommunity.microsoft.com/t5/microsoft-security-and/alert-new-blog-series-automation-in-cloud-app-security/ba-p/1608357)" or "[auto-triage infrequent country alerts](https://techcommunity.microsoft.com/t5/microsoft-security-and/auto-triage-infrequent-country-alerts-using-mcas-amp-power/ba-p/1644980)" are documented as part of this TechCommunity blog series.

### Considerations and References of "Cloud App Security"

- Interaction, integration and options of MCAS with other services in "Microsoft 365" are numerous and plays a significant role. Check out [MCAS design diagram (by ManagedSentinel.com)](https://www.managedsentinel.com/2020/05/03/mcas-design/) to get an overview of those connections.
- Regular check of "[MCAS Changelog](https://docs.microsoft.com/en-us/cloud-app-security/release-notes)" should be made to be informed about changes in your tenant and new features or risk detection.
- MCAS allows to gain sensitive information about a user which includes files, used cloud apps and all activity logs from connected apps. Therefore you should verify with your "IT Compliance and Data Privacy Department" if [anonymizing of user data](https://docs.microsoft.com/en-us/cloud-app-security/cloud-discovery-anonymizer) is required (to protect user privacy). Currently this feature is limited to user and machine names.
- Service health can be monitored on the "[MCAS status page](https://status.cloudappsecurity.com/)" and should be integrated for notification of service issues or delays of detections
- [Daily operational tasks](https://docs.microsoft.com/en-us/cloud-app-security/daily-activities-to-protect-your-cloud-environment) for MCAS are documented by Microsoft.
- Consider [Microsoft's best practices of implementing MCAS](https://docs.microsoft.com/en-us/cloud-app-security/best-practices) in your environment
- Governance Actions (by activity policies) must be in accord with your Azure AD environment.
    - Example: Suspended user will be reactivated after next Azure AD Connect sync interval.
- MCAS API can be easily discovered and tested by using the [postman collection](https://github.com/richlilly2004/CloudAppSecurity/blob/master/Postman/Microsoft%20Cloud%20App%20Security.postman_collection.json) (from Rich Lilly).
- Office 365 App Connector needs at least one assigned licensed user even if you want to use it to collect all Azure AD events only.
- Implement a process and simulate [investigation of anomaly detection alerts](https://docs.microsoft.com/en-us/cloud-app-security/investigate-anomaly)!
- MCAS is very useful and efficient to monitor suspicious privileged tasks in Azure.
    - Elevated Global Admin permissions on Azure Management (as already mentioned in the sample) is one of the use cases which can be (as far as I know) only be monitored by MCAS. Sami Lamppu has written a [detailed blog post](https://samilamppu.com/2020/06/18/monitor-elevated-global-admin-account-usage/) about it!
- RBAC in MCAS doesn't allow assignment of roles to Azure AD groups (only users are supported).
- [Scoped deployment](https://docs.microsoft.com/en-us/cloud-app-security/activity-privacy) can be very useful in setting up ["Proof-of-Concept" environment](https://gallery.technet.microsoft.com/Cloud-App-Security-Proof-4a49049f) or staged roll-outs in production
- Currently, blocking access to Cloud apps by (custom) indicators in "Microsoft Defender for Endpoint" (MDE) has some limitations:
    - MDE allows 15.000 indicators per tenant
    - This feature is supported on Windows 10 only (no support for ATP on MacOS or Linux yet)
- Detailed training on MCAS features are available as “[Ninja Training](https://techcommunity.microsoft.com/t5/microsoft-security-and/the-microsoft-cloud-app-security-mcas-ninja-training-is-here/ba-p/1877343)”

### Considerations and References of Microsoft Defender for Identity (MDI)

- Check alerts for false-positive events ("DCSync Attack") of "Azure AD Connect" server (exclude them for this specific detection).
- Signature-based capabilities can be evaluated as part of the "[Defender for Identity security alert lab](https://docs.microsoft.com/en-us/defender-for-identity/playbook-lab-overview)". Simulation of "Lateral Movement Attacks" is recommended and described in the [blog post (by Derk van der Woude)](https://medium.com/@derkvanderwoude/microsoft-defender-for-identity-lateral-movement-b55046c09870).
- By default, some domains are excluded from detections (Example: spotify.com)!
- Audit Policy of domain controllers must be configured to maximize detection capabilities. Using [this PowerShell script](https://github.com/microsoft/Azure-Advanced-Threat-Protection/tree/master/Auditing) should helps you to verify the configuration.
- [Review the known issues and limitations](https://docs.microsoft.com/en-us/defender-for-identity/troubleshooting-known-issues) of MDI sensors and detections
- Currently there's no option to collect Sensor health status!
- Global and Security Administrator are assigned with MDI "Administrator" permission by design. Default "Azure AD security groups" ("Azure ATP <InstanceName> Administrator/Users/Viewers") can be used to delegate permissions on the [MDI RBAC model](https://docs.microsoft.com/en-us/defender-for-identity/role-groups#types-of-product-short-security-groups). Modification of those group membership should be monitored for prevention of access to security sensitive logs or disabling the sensor/detection.
- I can only recommend to review the list of ["sensitive accounts and groups"](https://docs.microsoft.com/en-us/defender-for-identity/sensitive-accounts) and add non-builtin privileged objects (incl. hybrid identity components such as "Azure AD Connect" and the service accounts).
- Subscribe the RSS feed of "[What's new](https://docs.microsoft.com/en-us/defender-for-identity/whats-new)" to be notified about product changes and new features
- Service health can be monitored on the [MDI status page](https://health.atp.azure.com/) and integrated for notification of service issues or delays of detections. Consider to actively monitor the sensors in your infrastructure.
- [Sizing tool](https://aka.ms/aatpsizingtool) and [capacity planing guidance](https://docs.microsoft.com/en-us/defender-for-identity/capacity-planning) of MDI should be used in the planning phase to calculate amount of traffic, supportability and resource recommendations for sensors.

## Microsoft 365 Defender: Unified SecOps of M365 Services

*Sample use case: SecOps needs a unified visibility of logs and possibility of hunting across all "Microsoft 365" services and assets (data, identity, endpoints and cloud apps). Consolidated view on logs and summarized incidents from all "M365 Defender" services with enriched data is needed. Using centralized investigation of recorded activities (telemetry) in M365 services and empowering built-in (auto)-remediation of incidents by "Automated Investigation and Response (AIR) System".* 

![../2020-12-16-identity-security-monitoring/AzIdentity_AzMonitor.png](../2020-12-16-identity-security-monitoring/AzIdentity_MTP.png)

Microsoft 365 Defender (formerly "Microsoft Threat Protection") supports various services from the "M365 platform". Check the "[supported services](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/deploy-supported-services?view=o365-worldwide)" list to understand which data sources can be integrated. Start using "M365 Defender" is very easy by "[turn on](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/mtp-enable?view=o365-worldwide)" the described setting in the "Microsoft 365 Security Center".

### Data Sources in "M365 Defender"

#### IaaS/PaaS (Cloud and on-Premises) and "M365 Defender"

Azure resource-level or collected logs by Azure Monitor are *not* covered by "M365 Defender".
    Example: Event logs of Azure/Hybrid Servers or alerts from Azure Defender are *not* visible for hunting in this portal. 

#### Cloud Identity (Azure Active Directory) in "M365 Defender"

[Incidents](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/investigate-incidents?view=o365-worldwide) / [AlertInfo](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-alertinfo-table?view=o365-worldwide): Risk detections from "Azure AD Identity Protection" seems to be *not* included in the "Incidents" and aren't visible in the table "[AlertInfo](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-alertinfo-table?view=o365-worldwide)".
MCAS feeds many type of alerts to "M365 Defender" and "sign-in risk events" (detected by MCAS) is one of them. MCAS will be named as "ServiceSource" and "DetectionSource" in the log entries. Nevertheless, alerts from the "Risky sign-in" policy in MCAS, which collects also all "Identity Protection" risks (detected by "AAD Identity Protection"), can *not* be founded in "M365 Defender".

Examples:

- "Password Spray" will be detected by "Identity Protection" and listed in MCAS as "Risky sign-in" (as you have seen in the screenshots of previous article section). But it is *not* visible in the "AlertInfo" table or as part of an Incident in "M365 Defender".
- "Impossible Travel" is detected by MCAS and will be shown in the "Identity Protection" blade of Azure AD. But this risk detection is also listed as MCAS alert ("Impossible travel activity") and will be shown in the "Incident" view of "M365 Defender" and exists in the "[AlertInfo](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-alertinfo-table?view=o365-worldwide)" table.

    ![../2020-12-16-identity-security-monitoring/AzIdentity_MTP_Incidents.png](../2020-12-16-identity-security-monitoring/AzIdentity_MTP_Incidents.png)
    _Alerts from MCAS (and MDI) will be summarized as "Incidents" in the "M365 Security Portal". Detections by "Azure AD Identity Protection" (e.g. Password Spray) are missing. Alerts from connected apps in MCAS are also not listed in this case (e.g. "Mass Download" from OneDrive or SharePoint) or custom activity alerts (e.g. Elevated GA to Azure Management in Azure Portal)._

    ![../2020-12-16-identity-security-monitoring/AzIdentity_MTP_MCASAlerts.png](../2020-12-16-identity-security-monitoring/AzIdentity_MTP_MCASAlerts.png)
    _In comparison with the list of alerts in MCAS, not existing alerts in "M365 Defender" are bordered in red._


[IdentityLogonEvents](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-identitylogonevents-table?view=o365-worldwide): Authentication events captured by MCAS will be stored in this table. This seems to covers only "sign-in events" to connected apps and are similar to the "Activity Logs" in the MCAS blade (filtered by "Successful or failed login-ins"). As already described, "non-interactive" logons or sign-ins by "service principal"/"managed identity" are *not* covered.

[IdentityInfo:](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-identityinfo-table?view=o365-worldwide) Account information from various identity sources (including "Active Directory" and "Azure AD") will be stored here, to enable build relation between user objects (e.g. ObjectID, "On-Premises SID" and "Cloud SID"). Other details such as DisplayName, ProxyAddress or Account Status are also included.

#### On-Premises Identity (Active Directory) in "M365 Defender"

It's important to know that data of "Microsoft Defender for Identity" (MDI) will only be shown in the "M365 Defender" portal if the integration between MCAS and MDI is enabled. MCAS seems to be responsible to feeds the related MDI data to "M365 Defender".

Correlation of MDI alerts with other activities and alerts from "M365 Defender" services (such as "Microsoft Defender for Endpoint") gives you [new capabilities](https://techcommunity.microsoft.com/t5/microsoft-security-and/microsoft-365-defender-enriches-the-microsoft-defender-for/ba-p/1808275) to understand the context of Active Directory attacks. This becomes obvious if you think about the limited visibility of endpoint (threats) in MCAS.

Recently, Microsoft added the opportunity to use "Advanced Hunting" based on [events captured by MDI](https://techcommunity.microsoft.com/t5/microsoft-365-defender/hunt-for-threats-using-events-captured-by-azure-atp-on-your/ba-p/1598212). Another benefit (compared to MDI queries in the MCAS portal):
Writing KQL queries for advanced hunting in "M365 Security Portal" by using the advanced logs from the following tables:

[AlertInfo](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-alertinfo-table?view=o365-worldwide): Alerts from MDI will be stored in this table. AlertId includes the prefix "aa" (stands for Azure ATP?) followed by the original "AlertId" from the MDI portal (*not* the MCAS AlertID!).

[IdentityInfo:](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-identityinfo-table?view=o365-worldwide) As already described before, this table helps to correlate and build relation between various account objects. In this case, very helpful for advanced hunting and queries between "Azure AD" and "Active Directory" user objects.

[IdentityLogonEvents](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-identitylogonevents-table?view=o365-worldwide): Authentication events to your "Active Directory" will be stored in this table. The logon events will be sourced from the connected MDI instance in MCAS and shows similar results to a filtered "Activity Log" (by "Active Directory" app in the MCAS portal). Various types of logon events in "Active Directory" are covered (including Remote Desktop, Interactive and Credentials validation via NTLM/Kerberos). Failure reason of "sign-in attempts" are also included (e.g. OldPassword).

[IdentityQueryEvents](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-identityqueryevents-table?view=o365-worldwide): Queries on Active Directory objects (such as LDAP, DNS and SAMR) are collected from MDI in this table.

[IdentityDirectoryEvents](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-identitydirectoryevents-table?view=o365-worldwide): This table contains many identity-related (on-premises) audit and system events from the domain controller. User-level auditing of password or group memberships are included but also "domain controller events" such as PowerShell execution, Task scheduling or potential lateral movement.

#### Cloud Sessions (Microsoft Cloud App Security) in "M365 Security"

Cloud Discovery and activity logs from connected apps are *not* available for hunting in "M365 Defender".

[DeviceNetworkEvents](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-devicenetworkevents-table?view=o365-worldwide): As already described, "Microsoft Defender for Endpoints" (MDE) can be configured to forward signals to MCAS (for "Cloud Discovery" and "Visibility of (un)sanctioned cloud apps"). This table helps to start queries on the raw logs from MDE.

#### Collaboration Platforms (Office 365 Services)
[AlertInfo](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-alertinfo-table?view=o365-worldwide): Threat protection signals and data will be correlated from "Microsoft Defender for Office 365" (MDO) in M365 Defender. But it's limited to features and alerts around "Exchange Online" (such as "Safe Links" or Attachments).

Other Exchange Online-related logs and events are stored in the following tables and could be relevant for hunting of [phishing mails](https://docs.microsoft.com/en-us/windows/security/threat-protection/intelligence/phishing):

- [EmailEvents](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-emailevents-table?view=o365-worldwide) (e-mail delivery and blocking events)
- [EmailAttachmentInfo](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-schema-tables?view=o365-worldwide) (file attachment)
- [EmailPostDeliveryEvents](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-emailpostdeliveryevents-table?view=o365-worldwide) (security event after e-mail delivery)
- [EmailUrlInfo](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-emailurlinfo-table?view=o365-worldwide) (URLs / Safe Attachment in E-Mails)

[CloudAppEvents](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-cloudappevents-table?view=o365-worldwide): Activities from "Exchange Online" and "Microsoft Teams" (monitored by MCAS) are available for hunting here. Other O365 services are not supported yet! OneDrive and SharePoint Online will be [introduced in early 2021](https://techcommunity.microsoft.com/t5/microsoft-365-defender/hunt-across-cloud-app-activities-with-microsoft-365-defender/ba-p/1893857) and the existing "AppFileEvents" will be replaced at this time.

#### Device / Endpoint Security (Microsoft Defender for Endpoint) and "M365 Defender"

[Integration of "Microsoft Defender for Endpoint"](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/deploy-supported-services?view=o365-worldwide#limited-deployment-scenarios) (MDE) enables visibility on endpoint states, raw events, detections and alerts (which includes EDR/AV/attack surface reduction) and entities related to devices in M365 Defender.

[AlertInfo](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-alertinfo-table?view=o365-worldwide): Alerts from MDE will be shown in this table. Other events from this source will be stored in the following tables:

- [DeviceEvents](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-deviceevents-table?view=o365-worldwide) (security controls such as AV or Exploit protection)
- [DeviceLogonEvents](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-devicelogonevents-table?view=o365-worldwide) (logon events on/to devices from local or (Azure) AD users)
- [DeviceInfo](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-deviceinfo-table?view=o365-worldwide) (similar to the approach of the table "IdentityInfo", helps to correlate or build relation based on meta information by devices)

M365 Defender provides a "[Device profile page](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/device-profile?view=o365-worldwide)" which is accessible from the "Incident" view. This gives you a unified control on M365 Defender- and MDE-related actions.

### Analyze and Visualize with "M365 Defender"

#### Monitoring and Reporting ("Cards" in M365 Security Home)

Dashboard of "Microsoft 365 Security Center" allows to add "cards" for summarized reports of various sections of security areas in "M365 Defender" (including "[identity monitoring and reporting](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/monitor-and-report-identities?view=o365-worldwide)").

#### Investigation of Incidents in "M365 Defender"

Incidents can be managed in the portal by [adding comments, adjusting priority](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/manage-incidents?view=o365-worldwide), [reporting false/positives](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/mtp-autoir-report-false-positives-negatives?view=o365-worldwide) or [checking related entities (devices/users) or alerts](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/investigate-incidents?view=o365-worldwide#incident-overview).

[Suspicious entities](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/investigate-incidents?view=o365-worldwide#evidence) are also stored in the table "[AlertEvidence](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-alertevidence-table?view=o365-worldwide)" which can be used for custom queries or advanced hunting.

#### Advanced Hunting in "M365 Defender"

As already described, "M365 Defender" supports hunting on query-based analytics (KQL) across the various tables from supported M365 services. This allows you easily to start [hunting between activities and alerts of devices, e-mails and identities](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/advanced-hunting-query-emails-devices?view=o365-worldwide).

#### Custom Detections with "M365 Defender"

Advanced Hunting queries can be used to create a "[Detection Rule](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/custom-detection-rules?view=o365-worldwide)" for alerting.
This gives you the ability to proactively monitor specific critical events or potential threats. Applicable actions can be triggered if an entity is founded in the query (for example: Isolate device in case of a "Brute Force" attack).

### Integration and Response in "M365 Defender"

#### Auto-Investigation and Response (AIR)

M365 Defender supports only [remediation actions](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/mtp-remediation-actions?view=o365-worldwide) on suspicious or malicious "Devices" or "Emails". Pending (if approval is needed/configured) or completed actions are visible and can be managed in the "[Action Center](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/mtp-action-center?view=o365-worldwide)". This incident response activities [follows after an automated investigation](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/mtp-remediation-actions?view=o365-worldwide) by M365 Defender.

[Automation level and scope for Endpoints](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/mtp-configure-auto-investigation-response?view=o365-worldwide#review-or-change-the-automation-level-for-device-groups) can be configured in the "Microsoft Defender Security Center" (MDE Portal). [Policies for Office 365 can be configured in the "M365 Security Center"](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/mtp-configure-auto-investigation-response?view=o365-worldwide#review-your-security-and-alert-policies-in-office-365).

#### Threat Experts

Advice by Microsoft's Threat Experts can be requested directly from the "Incident" view.
This is an additional service which can be [enrolled for a 90-day-trial or on-Demand subscription (by Microsoft)](https://docs.microsoft.com/en-us/windows/security/threat-protection/microsoft-defender-atp/microsoft-threat-experts#before-you-begin). 

### Considerations and References of "M365 Defender"

- [Microsoft 365 Defender Ninja Training](https://techcommunity.microsoft.com/t5/microsoft-365-defender/become-a-microsoft-365-defender-ninja/ba-p/1789376) is a great resource to learn more!
- [Advanced Hunting Cheat Sheet](https://github.com/MiladMSFT/AdvHuntingCheatSheet/blob/master/MTPAHCheatSheetv01-light.pdf) gives some good samples and uses cases of queries across the supported services in "M365 Defender"
- [Samples of Advanced Hunting Queries](https://github.com/microsoft/Microsoft-365-Defender-Hunting-Queries) are available on GitHub and are ready to be used!
- Evaluate the "Attack Simulator" in the "M365 Security Center" to simulate attacks (such as phishing attacks) and start "security awareness" training for end-users
- Regular check on updates and changes in "[What's new in M365 Defender](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/whats-new?view=o365-worldwide)" or the message center in "Microsoft 365 admin center" is strongly recommended!
- Get an overview of the "M365 Defender" core features by [Microsoft's "educational training videos"](https://techcommunity.microsoft.com/t5/microsoft-365-defender/short-amp-sweet-educational-videos-on-microsoft-365-defender/ba-p/1525296)
- Custom Detection Rules can be configured only with a frequency between 1-24 hours and  built-in action as auto-response
- [Default permissions](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/mtp-permissions?view=o365-worldwide) in "M365 Defender" should be considered (to know who has access)
- Take a look on the "[Interactive guide](https://mslearn.cloudguides.com/en-us/guides/Protect%20your%20organization%20with%20Microsoft%20Threat%20Protection)" to get an overview about the hunting and incident management capabilities
- [Microsoft 365 Defender trial lab](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/mtp-evaluation?view=o365-worldwide) can be helpful to simulate attacks and learn the ways to resolve incidents or start advanced hunting.
- Public Preview of [M365 Defender APIs](https://docs.microsoft.com/en-us/microsoft-365/security/mtp/api-overview?view=o365-worldwide) allows integration and advanced hunting/queries programmatically

## Azure Sentinel: “Single pane of glass” across Azure, Microsoft 365 and 3rd party (cloud) platforms

*Sample use case: SecOps that needs a security visibility across all "Microsoft Cloud services" (Azure and M365) and (Hybrid/On-Premises) infrastructure. Extended possibilities for customization of auto-response, integration of "3rd party security tools" or implementation custom detections are required. Azure Sentinel empowers SIEM capabilities as part of a cloud-native and integrated security solution by Microsoft. Longer data retention of logs and alerts, out-of-the-box detections and visualization are further advantages.*

![../2020-12-16-identity-security-monitoring/AzIdentity_AzSentinel.png](../2020-12-16-identity-security-monitoring/AzIdentity_AzSentinel.png)

### Data Sources of "Azure Sentinel"

All of the following data sources can be connected to "Azure Sentinel" by data connectors.
Alerts from the Microsoft Security products can be [created as "Incident" automatically](https://docs.microsoft.com/en-us/azure/sentinel/create-incidents-from-alerts) which is strongly recommended to have been implemented (for unified incident view). Incidents generated by this products will be stored in the "[SecurityIncident](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/securityincident)" table of the workspace.

Most of the following features can be used to visualize or extend the logs and alerts from data sources:

- KQL-based Dashboards ("[Workbooks](https://docs.microsoft.com/en-us/azure/sentinel/quickstart-get-visibility#use-built-in-workbooks)")
- KQL-based Detections ("[Analytic Rules](https://docs.microsoft.com/en-us/azure/sentinel/tutorial-detect-threats-built-in)") and [Hunting Queries](https://docs.microsoft.com/en-us/azure/sentinel/hunting#get-started-hunting)
- Logic Apps for automated response or integration ("[Playbooks](https://docs.microsoft.com/en-us/azure/sentinel/tutorial-respond-threats-playbook)")

*Note: Most of the KQL queries and dashboards are already integrated as "Templates" and available in your instance (right after the deployment).
Nevertheless, I have added the links to the GitHub repository where you can find the original source of the templates.*

### IaaS/PaaS (Cloud and on-Premises) in "Azure Sentinel"

Azure Sentinel is built and will be deployed on "top of" Log Analytics Workspaces.
Collection of security and audit logs from "Azure Resources" or servers (On-Premises or hosted by "3rd Party Cloud Providers") can be implemented, alongside of the previously described "Azure Monitor (Logs)" integration. Azure Sentinel offers some "data connectors" to easily integrate the following data sources:

- [Data from "Azure Activity Logs"](https://docs.microsoft.com/en-us/azure/sentinel/connect-azure-activity): Selected subscriptions will stream their platform logs to the table "[AzureActivity](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/azureactivity)".
    - Insights of all operations and events within the "connected subscriptions" will be visualized by the built-in "Azure Activity" workbook. This gives you an overview about detected failures, warnings and caller activities to the "Azure Platform".
    - Eight different analytic rules are available for this "data source" including detection of anomalous privileged access such as "[Suspicious Resource Deployment or granting of permission](https://github.com/Azure/Azure-Sentinel/tree/master/Detections/AzureActivity)"
    - Security-related "diagnostic logs" should be also forwarded to the workspace of Azure Sentinel. It allows you to use "Analytic rules" for detecting unfamiliar access activities e.g. [in "Azure Key Vault" (mass secret retrieval or sensitive operations)](https://github.com/Azure/Azure-Sentinel/tree/master/Detections/AzureDiagnostics).
    - Other Cloud Providers: [Amazon Web Services (AWS)](https://docs.microsoft.com/en-us/azure/sentinel/connect-aws) can be integrated to stream all "Management Events" to Azure Sentinel as well.
- Data from [Syslog](https://docs.microsoft.com/en-us/azure/sentinel/connect-syslog) / [Security Event](https://docs.microsoft.com/en-us/azure/sentinel/connect-windows-security-events) Logs: This allows you to collect all security events from Windows and Linux machines by the "Log Analytics Agent". Scope configuration of streamed events (All Events, Common and Minimal) can be configured as previously described in this blog post. This settings also defines the amount and scope of "raw logs" that will be stored in the tables "[SecurityEvent](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/securityevent)" and "[Syslog](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/syslog)".
    - Azure Sentinel includes "analytic rules" to detect [failed logon attempts or "Brute Force" attacks](https://github.com/Azure/Azure-Sentinel/tree/master/Detections/Syslog) to your Linux servers. But also many [detections for user management or (RDP) sign-in anomaly](https://github.com/Azure/Azure-Sentinel/tree/master/Detections/SecurityEvent) on Windows Servers are available. In total over 31 detection templates based on "(Windows) Security Events" and over 11 rules for "Syslog" can be enabled to use this "data source".
    - Azure Sentinel provides two [ML-based behavior analytics to detect anomalous logins via SSH and RDP](https://techcommunity.microsoft.com/t5/azure-sentinel/what-s-new-azure-sentinel-machine-learning-behavior-analytics/ba-p/1521988).
    - Many workbooks are available to visualize the raw logs from servers in your environment including "[Event Analyzer](https://github.com/Azure/Azure-Sentinel/blob/master/Workbooks/EventAnalyzer.json)"(for audit object, file system or registry of Windows) or "[Linux Machines](https://github.com/Azure/Azure-Sentinel/blob/master/Workbooks/LinuxMachines.json)" (for insights of Linux events and errors).

### Azure Security Center (ASC) and "Azure Sentinel"

[Alert Data from Azure Security Center (detected by Azure Defender):](https://docs.microsoft.com/en-us/azure/sentinel/connect-azure-security-center) This connector allows to ingest alerts of detected threats by Azure Defender. The alerts will be found in the "[SecurityAlert](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/securityalert)" table as ProviderName "Azure Security Center". 

- Azure Sentinel includes a [unified workbook to get compliance, posture management and protection status](https://techcommunity.microsoft.com/t5/azure-sentinel/gain-compliance-posture-and-protection-insights-with-this-azure/ba-p/1290454) (including trends) on your "Azure Security Center" data.
- Playbooks allows you to [connect new "Azure Subscriptions" automatically](https://github.com/Azure/Azure-Sentinel/tree/master/Playbooks/AutoConnect-ASCSubscriptions) on a scheduled basis or to automate the notification of [incidents (to RBAC assigned Owners)](https://github.com/Azure/Azure-Sentinel/tree/master/Playbooks/Notify-ASCAlertAzureResource) or [recommendation (incl. IAM configuration) of resources](https://github.com/Azure/Azure-Sentinel/tree/master/Playbooks/Get-ASCRecommendations).

### Cloud Identity (Azure Active Directory) in "Azure Sentinel"

[Data from Azure AD Logs](https://docs.microsoft.com/en-us/azure/sentinel/connect-azure-active-directory): Audit and Sign-ins will be collected but *no* sign-in logs of "Service Principals", "Non-Interactive" or "Managed Identities" are covered yet. Therefore you have to configure the "[diagnostic settings](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/howto-integrate-activity-logs-with-log-analytics#send-logs-to-azure-monitor)" in the "Azure AD blade" manually to gather all insights of sign-in events to the workspace of Azure Sentinel.

- Many [KQL queries (analytic rules) are able to detect malicious activities such as "OAuth app" registrations](https://github.com/Azure/Azure-Sentinel/tree/master/Detections/AuditLogs). Analyses of sign-in attempts supports you to find [initial access attacks such as brute force to Azure Portal or Azure AD PowerShell anomalies](https://github.com/Azure/Azure-Sentinel/tree/master/Detections/Syslog).
- Over 34 "out of the box" hunting queries related to Azure AD "[Audit](https://github.com/Azure/Azure-Sentinel/tree/master/Hunting%20Queries/AuditLogs)" or "[Sign-in](https://github.com/Azure/Azure-Sentinel/tree/master/Hunting%20Queries/SigninLogs)" logs are available and empowers "Security Analysts" to detect e.g. anomalous activities in "account creation", "login to devices", "role assignments" or "rare privileged account activity". Hunting queries using multiple data sources alongside of Azure AD logs such as "Azure Activity Logs" or the "Behavior Analytics" from Azure Sentinel.
- Three different workbooks for "Azure AD" are included as "templates" for visualization. Insights about Azure AD-related logs will be visualized or partly included in combined views in other workbooks.

    ![../2020-12-16-identity-security-monitoring/AzIdentity_AzSentinelWorkbookAAD.png](../2020-12-16-identity-security-monitoring/AzIdentity_AzSentinelWorkbookAAD.png)
    *This workbook shows unified events between activity logs from Azure but also audit and sign-ins logs from Azure AD.*

- Auto-response on detected identity risks or threat detections can be extended by Playbooks. Microsoft offers samples for many actions such as [prompt user for investigation details](https://github.com/Azure/Azure-Sentinel/tree/master/Playbooks/Prompt-User), [isolate device of the user (by MDE)](https://github.com/Azure/Azure-Sentinel/tree/master/Playbooks/Isolate-MDATPMachine) or [revoke the sign-in session (token) by Microsoft Graph API](https://github.com/Azure/Azure-Sentinel/tree/master/Playbooks/Revoke-AADSignInSessions).

[Security Alerts from "Azure AD Identity Protection"](https://docs.microsoft.com/en-us/azure/sentinel/connect-cloud-app-security): All risk detection will be stored in the "[SecurityAlert](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/securityalert)" table under ProviderName "IPC" (= Identity Protection) by using this connector.

- Advanced correlation between incidents of [unfamiliar and atypical detections](https://github.com/Azure/Azure-Sentinel/tree/master/Detections/SecurityAlert) are possible with analytic rules.
- Status management of risky users ([confirm](https://github.com/Azure/Azure-Sentinel/tree/master/Playbooks/Confirm-AADRiskyUser) or [dismiss](https://github.com/Azure/Azure-Sentinel/tree/master/Playbooks/Dismiss-AADRiskyUser)) can be automated by Playbooks.

### On-Premises Identity (Active Directory) in "Azure Sentinel"

[Alerts from Microsoft Defender for Identity (MDI)](https://docs.microsoft.com/en-us/azure/sentinel/connect-azure-atp): Connector is listed as "Azure Advanced Threat Protection (Preview)" and forward the alerts to the "[SecurityAlert](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/securityalert)" table ("ProviderName" is named like the previously product name).

- Raw or detailed logs from MDI are *not* available in Azure Sentinel yet. Microsoft started to implement the ["Microsoft 365 Defender" connector](https://techcommunity.microsoft.com/t5/azure-sentinel/what-s-new-microsoft-365-defender-connector-now-in-public/ba-p/1865651) that seems to enable streaming of advanced logs in the future.

    ![../2020-12-16-identity-security-monitoring/AzIdentity_AzSentinelM365Connector.png](../2020-12-16-identity-security-monitoring/AzIdentity_AzSentinelM365Connector.png)

    *"Microsoft 365 Defender" connector allows to stream the already known "Advanced hunting" tables (with raw event data) from the "M365 Security Portal" to Azure Sentinel. Currently this is limited to "Defender for Endpoint" but seems to be coming for other M365 Defender products as well!*

    But at this time, KQL-based queries and hunting (on detailed logs) are useable in "M365 Defender" only.

- Collected (security) logs from domain controllers (via Log Analytics Agent / Azure Security Center) can be used to gain insights of the on-premises environment. Workbooks to analyze security events to [detect usage of insecure protocols (NTLMv1, WDigest)](https://techcommunity.microsoft.com/t5/azure-sentinel/azure-sentinel-insecure-protocols-workbook-reimagined/ba-p/1558375) or visualize anomalies and user activities across "Identity & Access" operations are available.

    ![../2020-12-16-identity-security-monitoring/AzIdentity_AzSentinelWorkbookIAM.png](../2020-12-16-identity-security-monitoring/AzIdentity_AzSentinelWorkbookIAM.png)
    *Workbook template "Identity & Access" uses logs from the "SecurityEvents" table to visualize authentication events and user activities in your "Active Directory" environment.*

### Cloud Sessions (Microsoft Cloud App Security) in "Azure Sentinel"

[Data from Cloud App Security](https://docs.microsoft.com/en-us/azure/sentinel/connect-cloud-app-security): All alerts from MCAS will be stored in the table "[SecurityAlert](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/securityalert)". The second data type of the connector collects the "Discovery Log" ("Shadow IT" reports) from MCAS to the "[McasShadowItReporting](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/mcasshadowitreporting)" table in the Sentinel workspace.

- Azure Sentinel integration will be also [configured in the "MCAS portal"](https://docs.microsoft.com/en-us/cloud-app-security/siem-sentinel) and allows to specify filters on the discovery logs.
- [Data schema of the MCAS logs](https://docs.microsoft.com/en-us/cloud-app-security/siem-sentinel#alerts-and-discovery-logs-in-azure-sentinel) in Azure Sentinel are also documented by Microsoft.
- Discovery Logs in Azure Sentinel can be visualized by using the [pre-configured "Workbook"](https://github.com/Azure/Azure-Sentinel/blob/45310fe6be3cb30cbd8f073443a0c2f9144115fa/Workbooks/MicrosoftCloudAppSecurity.json) template.
- Resolving MCAS alerts can be automated as part of Playbook (instead of PowerAutomate) in Azure Sentinel.
    - Example:  [Closing alert of "Infrequent Country"](https://github.com/Azure/Azure-Sentinel/tree/master/Playbooks/Resolve-McasInfrequentCountryAlerts) if affected users meet a criteria such as "out-of-office status" or "group membership" (employees who travel internationally). Details of this example are very well [explained by Sebastian Molendijk in his video](https://www.youtube.com/watch?v=ql8x4rC6m9A).

### Collaboration Platforms (Office 365 Services) in "Azure Sentinel"

[Data from Office 365 Logs](https://docs.microsoft.com/en-us/azure/sentinel/connect-office-365): Activity logs from SharePoint, Exchange and Teams will be stored in the "[OfficeActivity](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/officeactivity)" table by this connector.

- User activities by these three workloads can be visualized in the ["Office 365" workbook](https://github.com/Azure/Azure-Sentinel/blob/master/Workbooks/Office365.json).
- Many analytic rules are available [to detect suspicious "Office 365" activities](https://github.com/Azure/Azure-Sentinel/tree/master/Detections/OfficeActivity) but also some hunting queries on [Exchange Online mailboxes, SharePoint downloads](https://github.com/Azure/Azure-Sentinel/tree/master/Hunting%20Queries/OfficeActivity) or [Teams activities](https://github.com/Azure/Azure-Sentinel/tree/master/Hunting%20Queries/TeamsLogs) are available.
- Other compliance, operational or security logs from "Office 365" (such as Message Trace logs) are *not* included. Collecting the missing logs are described in a [TechCommunity blog post and will be achieved trough Azure Logic Apps](https://techcommunity.microsoft.com/t5/azure-sentinel/how-to-protect-office-365-with-azure-sentinel/ba-p/1656939).

[Alerts from "Microsoft Defender for Office 365" (MDO)](https://docs.microsoft.com/en-us/azure/sentinel/connect-office-365-advanced-threat-protection): Data connector is named as "Office 365 Advanced Threat Protection" (OATP) and allows to store many types of alerts from the "Office Security and Compliance Center"  to the "[SecurityAlert](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/securityalert)" table.

Advanced logs (data) and all types of alerts from MDO are *not* available yet. As already mentioned,  the "[M365 Defender Connector](https://docs.microsoft.com/en-us/azure/sentinel/connect-microsoft-365-defender)" seems to improve the log integration between MDO and Sentinel in the future.

### Device / Endpoint Security (Microsoft Defender for Endpoint) in "Azure Sentinel"

[Alerts from "Microsoft Defender for Endpoint" (MDE)](https://docs.microsoft.com/en-us/azure/sentinel/connect-microsoft-defender-advanced-threat-protection): Data connector to fetch alerts generated by endpoint protection is also named by the old brand "Microsoft Defender Advanced Threat Protection" (MDATP). Alerts will be also stored (similar to the other M365 Defender products) in the "[SecurityAlert](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/securityalert)" table.

- Many [playbooks](https://github.com/Azure/Azure-Sentinel/tree/master/Playbooks) are available from Microsoft to use MDE to restrict entities (block IP address, URL, app execution,...) or further interaction (get investigation package or list of the Threat & Vulnerability Management) as part of an automated response process.

[Data from M365 Defender (Device*)](https://docs.microsoft.com/en-us/azure/sentinel/connect-microsoft-365-defender): Advanced logs from the already known "advanced hunting" tables (DeviceInfo, DeviceLogonEvents,...) in "M365 Defender" will be streamed to "Azure Sentinel" by this new connector.

- This allows new hunting and correlation options between logs that can be only collected from Azure Monitor/Azure Sentinel and M365 integrated logs.
    - Example: "Windows Sign-in events" ("SigninLogs" table, sourced from Azure AD) can be correlated natively with entries of the "DeviceLogonEvents" table which covers local sign-in and authentication events from MDE.
- M365 Defender and Azure Sentinel are using KQL as query language. Therefore all your existing hunting queries from MDE/M365 Defender should be easily used in Azure Sentinel as well.

## Investigation of Incidents in "Azure Sentinel"

### Incidents and Workbooks in "Azure Sentinel" blade

Incidents blade of "Azure Sentinel" shows all alerts from the "connected data sources" in Azure Sentinel. This includes MCAS "custom alerts" (e.g. activity policy "Elevated Global Admin to Azure Management") and all other built-in policies or detections (e.g. "Mass Download by a single user" in MCAS). But also alerts from "Azure Security Center" ("Access from a Tor exit") and "Analytic rules" (from Azure Sentinel) on Azure Activity Logs ("Rare subscription-level operations") will be listed.

![../2020-12-16-identity-security-monitoring/AzIdentity_AzSentinelIncidents.png](../2020-12-16-identity-security-monitoring/AzIdentity_AzSentinelIncidents.png)

It's important to understand the difference between incident and alerts in Azure Sentinel:
Incidents are a group of related alerts and will be correlated by Azure Sentinel.

- As already described, alerts from connected security products (MDE, MDI, Azure Defender, etc.) are only displayed as "Incident" if "[Microsoft Security Incident Creation Analytic Rules](https://docs.microsoft.com/en-us/azure/sentinel/create-incidents-from-alerts)" are configured in the "Analytics" blade.
- Built-in (templates) or custom analytic rules can be grouped as "Incident" if an alert is triggered (enabled by default).

It is also important to know the [three different types of analytic rules](https://docs.microsoft.com/en-us/azure/sentinel/tutorial-detect-threats-built-in#about-out-of-the-box-detections) and the logic behind them.

Templates of various workbooks are included that gives you an advanced view of incidents:

- [Incident Overview](https://github.com/Azure/Azure-Sentinel/blob/master/Workbooks/IncidentOverview.json) (In-depth information to a specific incident case)
- [Investigation Insights](https://github.com/Azure/Azure-Sentinel/blob/master/Workbooks/InvestigationInsights.json) (timeline and trend of incidents combined with detailed information about entities)
- [MITRE ATT&CK](https://github.com/Azure/Azure-Sentinel/blob/master/Workbooks/MITREAttack.json) (Visualizing coverage of "MITRE ATT&CK" framework on Azure Sentinel)

### Investigation Graph

Investigation between security events based on "device" or "user" detections but also from "cloud" or "on-premises" resources can be hard. Sentinel offers a visualization of raw data and timeline to increase the visibility of context and helps to start your investigation on relation between entities of the incidents.
Therefore, [Investigation graph](https://docs.microsoft.com/en-us/azure/sentinel/tutorial-investigate-cases) can be very useful for investigate your incidents.

Recently, Microsoft introduced the "[Entity Insights](https://techcommunity.microsoft.com/t5/azure-sentinel/what-s-new-entity-insights-for-convenient-investigation-checks/ba-p/1801496)" feature which shows detailed information of the related entities in the "investigation graph".

![../2020-12-16-identity-security-monitoring/AzIdentity_AzSentinelInvestigationGraph.png](../2020-12-16-identity-security-monitoring/AzIdentity_AzSentinelInvestigationGraph.png)

*Investigation starts on "Mass Download" incident and exploring all other related alerts from the entities. At the end, a comprehensive attack timeline and visualized progression of events will be shown. Detections of "Brute-Force" against "Active Directory" and the "Azure Portal" can be analyzed in the one investigation step.* 

### Advanced multistage attack detection

[Advanced multistage attack detection](https://docs.microsoft.com/en-us/azure/sentinel/fusion) is based on machine learning (["Fusion" technology](https://www.microsoft.com/security/blog/2020/02/20/azure-sentinel-uncovers-real-threats-hidden-billions-low-fidelity-signals/)) and automates the correlation on various types of attack scenarios. This includes data exfiltration, lateral movement and malicious administrative activities. 

![../2020-12-16-identity-security-monitoring/AzIdentity_AzSentinelFusion.png](../2020-12-16-identity-security-monitoring/AzIdentity_AzSentinelFusion.png)
*Fusion detects a multistage attack and build an incident with collections of related alerts.*

### Entity Behavior

[User Entity Behavior Analytics (UEBA)](https://techcommunity.microsoft.com/t5/azure-sentinel/guided-ueba-investigation-scenarios-to-empower-your-soc/ba-p/1857100) allows investigation of entities (such as user or devices) and their behavior based on Azure Sentinel data.

- Onboarded data sources and their raw data will be analyzed by the "UEBA Engine" in Azure Sentinel to find anomalies.
- User information will be synchronized from "Azure AD" to enrich user profiles in the UEBA entity pages.
- Details on the architecture and engine to [identify advanced threats with this feature](https://docs.microsoft.com/en-us/azure/sentinel/identify-threats-with-entity-behavior-analytics) are documented by Microsoft.

UEBA can be [enabled](https://docs.microsoft.com/en-us/azure/sentinel/enable-entity-behavior-analytics) from the "Entity behavior" blade in Azure Sentinel.
Selection of data sources (used by UEBA) can also be configured in this blade and includes "Azure AD" (Audit / Sign-in logs), "Azure Activity" and "Security Events" (from all connected Microsoft Security products). Scoring and timeline of the "Entity pages" are longer visible in comparison with the MCAS "user page".

Enriched events from the "UEBA engine" will be stored in the "[BehaviorAnalytics](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/behavioranalytics)" table and are readable as (table) entries [by using KQL queries](https://docs.microsoft.com/en-us/azure/sentinel/identify-threats-with-entity-behavior-analytics#querying-behavior-analytics-data). Microsoft is also using this table to visualize "UEBA results" in the workbook template "[User And Entity Behavior Analytics](https://github.com/Azure/Azure-Sentinel/blob/master/Workbooks/UserEntityBehaviorAnalytics.json)". The founded anomalies will be scored with "Investigation Priority Score" and mapped to the "MITRE ATT&CK" framework.

Analytics from "UEBA" based on accounts, IP addresses and hosts entities can be displayed in the "Entity behavior" blade.

![../2020-12-16-identity-security-monitoring/AzIdentity_AzSentinelUEBA.png](../2020-12-16-identity-security-monitoring/AzIdentity_AzSentinelUEBA.png)
*Entity pages shows an "Alerts and Activities Timeline" with all incidents by "Microsoft Security products" (generated by incident creation rule) or analytic rules (built-in or custom queries) and anomalous detections based on the behavioral learning in" UEBA". Insight box visualize anomalous activities and sign-in events from the various data sources.*

### Hunting Queries
Over 175 [hunting queries](https://docs.microsoft.com/en-us/azure/sentinel/hunting) are already integrated and can be used by "Security Analysts" to start hunting on various types of threats incl. "initial access" or "privilege escalation". The list of hunting queries can be filtered by "data sources" and "tactics". All queries are written in KQL and can be edited or customized.
New hunting queries can be created from the blade and Microsoft is [adding new "out of the box" queries](https://techcommunity.microsoft.com/t5/azure-sentinel/what-s-new-80-out-of-the-box-hunting-queries/ba-p/1892067) on a regular basis.

## Integration and Response in "Azure Sentinel"

### Playbooks

A logic app can be triggered to automate "threat response" when an "analytic rule" generates the alert.
All logic apps that includes "Azure Sentinel alert trigger" can be used as "Playbook".
Microsoft describes the configuration of this playbooks in one of the [Azure Sentinel tutorials](https://docs.microsoft.com/en-us/azure/sentinel/tutorial-respond-threats-playbook).
As already mentioned, many logic app templates are available from [GitHub](https://github.com/Azure/Azure-Sentinel/tree/master/Playbooks).
Monitoring of playbooks is an important part of daily operations (especially if it's an essential part of your incident and response process). [Workbook for "Playbook Health"](https://techcommunity.microsoft.com/t5/azure-sentinel/what-s-new-monitoring-your-logic-apps-playbooks-in-azure/ba-p/1873211) might be helpful to get an overview about failed runs and latency.

In addition, there is also a ["Logic App connector" for Azure Sentinel](https://docs.microsoft.com/en-us/connectors/azuresentinel/) which allows to update or use information from incidents.

### WatchList

Recently, Microsoft introduces [Watchlist](https://docs.microsoft.com/en-us/azure/sentinel/watchlists) as feature to collect data from "external sources" for correlation in the analytic rules. This enables also the possibilities of enrichment by external events or business data. One of Microsoft's playbook samples is using "WatchList" to close an [incident from known IP addresses](https://github.com/Azure/Azure-Sentinel/tree/ed03d35701b6a914c742bf0752f891b1c465c656/Playbooks/Watchlist-CloseIncidentKnownIPs). On this general approach as an example, you can use a playbook to ingest "Trusted IP addresses from Microsoft (backend)" to enrich false-positive events of unfamiliar locations.

### Notebooks

Azure Sentinel allows to use "Jupyter notebooks" running on "Azure Machine Learning" (AML) platform and using "Azure Sentinel" data. Notebook templates are already included such as an "[Entity Explorer for Account](https://github.com/Azure/Azure-Sentinel-Notebooks/blob/master/Entity%20Explorer%20-%20Account.ipynb)" or "[guided hunting on anomalous Exchange Online Sessions](https://github.com/Azure/Azure-Sentinel-Notebooks/blob/master/Guided%20Investigation%20-%20Anomaly%20Lookup.ipynb)".

Notebook are very powerful for [hunting of security threats](https://docs.microsoft.com/en-us/azure/sentinel/notebooks) and allows [enhancement of existing "Azure Sentinel data" by external threat intelligence](https://techcommunity.microsoft.com/t5/azure-sentinel/security-investigation-with-azure-sentinel-and-jupyter-notebooks/ba-p/432921).

## Considerations and References of "Azure Sentinel"

- [Azure Sentinel Ninja (Level 400)](https://techcommunity.microsoft.com/t5/azure-sentinel/become-an-azure-sentinel-ninja-the-complete-level-400-training/ba-p/1246310) Training includes a great list of resources to start your study on Microsoft's SIEM solution.
- Microsoft is offering many interesting webinars around Azure Sentinel! Check out the [upcoming events or the records of the past webinars](https://techcommunity.microsoft.com/t5/microsoft-security-and/security-community-webinars/ba-p/927888).
    - One of my favourite session is "[Tackling Identity](https://www.youtube.com/watch?v=BcxiY32famg)" because of the high proportion of use cases and live-demos (KQL samples) that shows the advantages and technical possibilities of Azure Sentinel.
- Consider [Microsoft's Best Practices for Azure Sentinel](https://www.microsoft.com/security/blog/wp-content/uploads/2020/07/Azure-Sentinel-whitepaper.pdf)
    - Best Practices for [designing Azure Sentinel and Azure Security Center workspaces](https://techcommunity.microsoft.com/t5/azure-sentinel/best-practices-for-designing-an-azure-sentinel-or-azure-security/ba-p/832574) is very important!
    - Consider to create as few workspaces as possible!
- [Azure Sentinel To-Go](https://techcommunity.microsoft.com/t5/azure-sentinel/azure-sentinel-to-go-part1-a-lab-w-prerecorded-data-amp-a-custom/ba-p/1260191?WT.mc_id=twitter-social-thmaure) is a great way to build a lab environment with prerecorded data.
- Comparison between "Azure Sentinel" and "M365 Defender" is one of the most common question. [Jan Geisbauer](https://www.linkedin.com/pulse/azure-sentinel-vs-microsoft-defender-jan-geisbauer/?trackingId=V%2B5QLIutQOOT7bHW7QUyVg%3D%3D) and [Sami Lamppu](https://samilamppu.com/2020/11/24/microsoft-365-defender-vs-azure-sentinel-which-one-to-use/) has written very good blog posts to give an general answer to this question.
- Maarten Goet has written a great blog post why ["VSCode" becomes essential for threat hunting with Azure Sentinel](https://medium.com/@maarten.goet/visual-studio-code-the-swiss-army-knife-for-threat-hunting-with-azure-sentinel-503e7ef38c96)
- [Enrichment with "Azure AD information"](https://techcommunity.microsoft.com/t5/azure-sentinel/enriching-azure-sentinel-with-azure-ad-information/ba-p/1288805) can be important to associate detailed information to "Azure Activity" or "Office 365" logs which is well described in this TechCommunity blog post.
- Enrichment of alert entities with other sources like MCAS or "Microsoft Defender for Endpoint" is also possible. A [template](https://github.com/Sebmolendijk/ARMLogicApps/tree/master/EntitiesEnrichment) and [detailed video](https://www.youtube.com/watch?v=YZr-New3yCI&feature=youtu.be) is available to build entity enrichment by a  playbook.
- Deployment of Azure Sentinel can be automated as part of a [centralized repository and CI/CD pipeline](https://techcommunity.microsoft.com/t5/azure-sentinel/deploying-and-managing-azure-sentinel-ninja-style/ba-p/1858073). This is a great approach if you have the need to manage staging or multi-tenant environments.
- Azure Lighthouse can be used for a multi-workspace view and [tracking an attack multiple tenants](https://techcommunity.microsoft.com/t5/azure-sentinel/using-azure-lighthouse-and-azure-sentinel-to-investigate-attacks/ba-p/1043899). Collection of Azure AD or O365 logs from different tenants is also possible if you [configure a custom data connector](https://techcommunity.microsoft.com/t5/azure-sentinel/o365-amp-aad-multi-tenant-custom-connector-azure-sentinel/ba-p/1848968).
- Monitoring of the Workspace Health is one the daily operational tasks in Azure Sentinel.
The workbook for [usage reporting](https://techcommunity.microsoft.com/t5/azure-sentinel/usage-reporting-for-azure-sentinel/ba-p/1267383) is essential to check latency, cost and table analysis (past, current and estimated size of tables) of your Log Analytics workspace / Sentinel instance.
- Microsoft offers a [workbook to verify the monitoring health](https://docs.microsoft.com/en-us/azure/sentinel/monitor-data-connector-health) in Azure Sentinel.
It's strongly recommended to use this integrated workbook to check the health of agents and "data connectors" but also the connectivity and performance within Azure Sentinel.
- You want to learn KQL? Check the [Basic KQL Course from Pluralsight](https://www.pluralsight.com/courses/kusto-query-language-kql-from-scratch) or "[Become a KQL Ninja" (KQL Internals Study Guide) by Huy](https://security-tzu.com/2020/08/07/become-a-kql-ninja/).
<br>
<br>
<br>
<span style="color:silver;font-style:italic;font-size:small">Original cover image by [200degrees / Pixabay](https://pixabay.com/vectors/dual-screen-programming-coding-1745705/)</span>
