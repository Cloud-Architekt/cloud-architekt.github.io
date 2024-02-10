---
title: "Microsoft Entra Workload ID - Advanced Detections and Enrichment in Microsoft Sentinel"
excerpt: "Collecting details of all workload identities in Microsoft Entra ID allows to build correlation and provide enrichment data for Security Operation Teams. In addition,
it also brings new capabilities for creating custom detections. In this blog post, I will show some options on how to implement a data source for enrichment of non-human identities to Microsoft Sentinel and the benefits for using them in analytics rules."
header:
  overlay_image: /assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetection.png
  overlay_filter: rgba(102, 102, 153, 0.85)
  teaser: /assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetection.png

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
last_modified_at: 2023-12-18
---

This blog post is part of a series about Microsoft Entra Workload ID:
- [Introduction and Delegated Permissions](https://www.cloud-architekt.net/entra-workload-id-introduction-and-delegation)
- [Lifecycle Management and Operational Monitoring](https://www.cloud-architekt.net/entra-workload-id-lifecycle-management-monitoring/)
- [Threat detection with Microsoft Defender XDR and Sentinel](https://www.cloud-architekt.net/entra-workload-id-threat-detection)
- [Advanced Detection and Enrichment in Microsoft Sentinel](https://www.cloud-architekt.net/entra-workload-id-advanced-detection-enrichment)
- Incident Response (coming soon)

## Integration of AzADServicePrincipalInsights as Custom Table

In the first part of the blog post, I’ve already described "[AzADServicePrincipalInsights](https://github.com/JulianHayward/AzADServicePrincipalInsights)" (AzADSPI) from [Julian Hayward](https://github.com/JulianHayward). The tool allows to collect richfull insights and tracking changes of Workload Identities and export them as JSON in a repository. But we can use also the data to ingest them to Microsoft Sentinel for enrichment. Therefore, I’ve created a pipeline which ingest the data to a Microsoft Sentinel Workspace by using PowerShell scripts.

_[**Update 2024-02-10**]: Option to ingest data to Log Analytics has been integrated to AzADSPI. Julian has also improved my previous script logic and has become part of the solution. I've updated the instruction to use the original repository instead of my forked version._

### Executing AzADServicePrincipalInsights in your GitHub repo

1. First of all, create a private project/repository with the name "AzADServicePrincipalInsights" in GitHub and clone the forked version of AzADSPI from the repository [JulianHayward/AzADServicePrincipalInsights](https://github.com/JulianHayward/AzADServicePrincipalInsights). This repository includes also the PowerShell script and workflow extensions to ingest the output to Microsoft Sentinel.
2. Create an app registration and add the [required permission](https://github.com/JulianHayward/AzADServicePrincipalInsights/tree/main#permissions) as Application Permission and Azure RBAC assignment. You can also create a user-assigned managed identity but in that case the Graph API permissions needs to be configured by API or PowerShell.

    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect0.png)

3. Follow the steps from the following Microsoft Learn article to configure Federated Credentials in GitHub: [Add federated credentials - Connect GitHub and Azure | Microsoft Learn](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Clinux#add-federated-credentials)
Make sure, that the values `CLIENT_ID`  , `TENANT_ID` and `SUBSCRIPTION_ID` has been added as repository secret.
4. Use the workflow template file ".github/workflows/AzADServicePrincipalInsights_OIDC.yml" and customize the parameters. It’s mandatory to change the `ManagementGroupId`.

    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect1.png)

5. Execute the workflow and verify that AzADSPI is able to collect the data and was able to commit the files successfully to the main branch.

### **Create Data Collection Endpoint and Rule to use Azure Monitor Ingest API**

Next, we will create the required data collection endpoint in [Azure Portal](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/tutorial-logs-ingestion-portal#create-data-collection-endpoint).
Those single configuration steps are well documented in Microsoft Learn. Therefore I’ve added the links to the related articles in the headlines and added some notes and configuration parameters which are important to know.

1. [Create data collection endpoint](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/tutorial-logs-ingestion-portal#create-data-collection-endpoint)
In my example, I’m using the same subscription as Microsoft Sentinel for creating a data collection endpoint and rule. Make sure that Data Collection Endpoint and Rule are created in the same but in a dedicated resource group.

    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect2.png){: width="70%" }

2. [Create new table in Log Analytics workspace](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/tutorial-logs-ingestion-portal#create-new-table-in-log-analytics-workspace)

    I’ve chosen the table name "AzADServicePrincipalInsights_CL" which will be used later in the parser and analytics rules:

    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect3.png)

3. [Parse and filter sample data](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/tutorial-logs-ingestion-portal#parse-and-filter-sample-data)
Get the script "[AzADSPI_DataCollectionIngest.ps1](https://github.com/Cloud-Architekt/AzurePrivilegedIAM/blob/WiBlogPost/Scripts/AzADSPI/AzADSPI_DataCollectionIngest.ps1)" from my repository and execute it by using the parameter `SampleDataOnly` which allows you to get a JSON output. Export the content as file and use them as sample data.
4. [Assign permissions to the DCR](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/tutorial-logs-ingestion-portal#assign-permissions-to-the-dcr)
Use the pre-created Service Principal or Managed Identity (with Federated Credential) which has been created previously for the role assignment. In this scenario, the GitHub workflow needs the privileges to ingest the data.

### **Implement Pipeline to ingest data to Microsoft Sentinel**

1. Next, go back to the workflow file "AzADServicePrincipalInsights_OIDC.yaml" for changing variable of `IngestToLogAnalytics` to `true` and editing the following variables with the corresponding values of your environment:
    - `ManagementGroupId`
    - `DataCollectionRuleSubscriptionId`
    - `DataCollectionRuleResourceGroup`
    - `DataCollectionRuleName`
    - `LogAnalyticsCustomLogTableName` (e.g., AzADServicePrincipalInsights_CL)
    - `ThrottleLimitMonitor` (default value can be keep as it is) 

        ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect4.png)

2. The logic for ingest the data to Log Analytics is defined in the PowerShell script "AzADSPI/AzADSPI_DataCollectionIngest.ps1" in the folder "pwsh". There's no further customizing needed by default.
3. Run the workflow and wait for 10-15 minutes to verify if the data has been successfully ingested. Check if any event entry exists on the defined table name (e.g. `AzADServicePrincipalInsights_CL`).
4. I’ve created a template for a KQL function with the name "AzADSPI" which will standardize the column names to my defined and preferred schema. This also supports me in sharing the same KQL query logic across other data sources that I’m using in my examples and detection queries. Follow the steps from Microsoft Learn article to [create and use functions in Microsoft Sentinel.](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/functions#use-a-function)

    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect5.png)


## Publish WatchList "WorkloadIdentityInfo" with SentinelEnrichment

Together with my colleague [Fabian Bader](https://www.notion.so/Microsoft-Entra-Workload-ID-4-6-Advanced-Detections-and-Enrichment-in-Microsoft-Sentinel-98bd997e1c3941999ca3f8ce77ad6adc?pvs=21), I have worked on an alternate way to publish enrichment data to Microsoft Sentinel. WatchLists are a great way to maintain a list of this kind of asset information which can be used in KQL queries within Microsoft Sentinel.

We have published a PowerShell module named "[SentinelEnrichment](https://www.powershellgallery.com/packages/SentinelEnrichment)" which automates the process to create and update the WatchList. This module can be executed in a GitHub action workflow, Automation Account, Azure Function or any other environment which supports PowerShell.

I’ve used some code from my EntraOps PoC project to gather various details about Workload Identities. The cmdlet can be found here and will provide the content for the WatchList which we like to upload in the following automation job. In this example, I will use an automation account for collecting the data and upload the WatchList to Sentinel.

*Side Note: This solution offers a different set of information compared to the AzADSPI. Some details such as Azure RBAC assignments or Delegated API Permissions are not part of the WatchList. The size of a row is limited and therefore just a subset can be provided in a single WatchList.*

### Create and prepare an Automation Account

1. [Create an automation account](https://learn.microsoft.com/en-us/azure/automation/automation-create-standalone-account?tabs=azureportal) and enable the [System-assigned Managed Identity](https://learn.microsoft.com/en-us/azure/automation/enable-managed-identity-for-automation#enable-a-system-assigned-managed-identity-for-an-azure-automation-account)
2. Add the required Microsoft Graph API application permissions to the Managed Identity by using Microsoft Graph PowerShell or any other Graph Client. I’ve customized the published sample by [Niklas Tinner](https://github.com/thenikk) which can be found [here](https://www.notion.so/Microsoft-Entra-Workload-ID-4-6-Advanced-Detections-and-Enrichment-in-Microsoft-Sentinel-98bd997e1c3941999ca3f8ce77ad6adc?pvs=21).

    ```jsx
    #Install required module
    Install-Module Microsoft.Graph -Scope CurrentUser

    #Connect to Graph
    Connect-MgGraph -Scopes Application.Read.All, AppRoleAssignment.ReadWrite.All, RoleManagement.ReadWrite.Directory

    #Insert Object ID of Managed Identity
    $ManagedIdentityObjectId = Read-Host -Prompt "Enter Object Id of the System-assigned Managed Identity on the Azure Resource"

    #Insert permissions of Graph
    $AppRoles = @("Application.Read.All","Group.Read.All", "RoleManagement.Read.Directory")

    #Find Graph permission
    $MsGraph = Get-MgServicePrincipal -Filter "AppId eq '00000003-0000-0000-c000-000000000000'"
    $Roles = $MsGraph.AppRoles | Where-Object {$_.Value -in $AppRoles}

    #Assign permissions
    foreach ($Role in $Roles) {
        New-MgServicePrincipalAppRoleAssignment -ServicePrincipalId $ManagedIdentityObjectId -PrincipalId $ManagedIdentityObjectId -ResourceId $MsGraph.Id -AppRoleId $Role.Id
    }
    Disconnect-MgGraph
    ```

3. In addition to Microsoft Graph, the automation accounts need also permission to write and delete WatchList(s) in the Microsoft Sentinel Workspace. Assign the managed identity the role "Microsoft Sentinel Contributor" on Resource Group-Level or create/use a custom RBAC role including the following least privileges:
    - Microsoft.SecurityInsights/Watchlists/read
    - Microsoft.SecurityInsights/Watchlists/write
    - Microsoft.SecurityInsights/Watchlists/delete
4. Add the SentinelEnrichment PowerShell Module by using the [import function from the PowerShell Gallery](https://learn.microsoft.com/en-us/azure/automation/shared-resources/modules#import-modules-from-the-powershell-gallery). Choose PowerShell 7.2 as runtime environment.

    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect23.png){: width="90%" }
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect24.png)

5. Verify that the module "SentinelEnrichment" has been successfully imported as module.

    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect6.png)


### Add and execute the runbook to create "WorkloadIdentityInfo"

1. Add details about the Sentinel Workspace in the [variables of the automation accounts](https://learn.microsoft.com/en-us/azure/automation/shared-resources/variables?tabs=azure-powershell#create-and-get-a-variable-using-the-azure-portal). Create variables under the section "Shared Resources" with the following names and the related values:

    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect7.png){: width="75%" }

    - SentinelResourceGroupName
    - SentinelSubscriptionId
    - SentinelWorkspaceName
2. Create a new runbook with runbook type "PowerShell" and Runtime version 7.2.

    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect8.png){: width="75%" }

3. Copy the content of the script from my repository and past them into the runbook:

    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect9.png)

4. Click on "Test pane" to validate that the script in association with the required permissions and variables works.
5. At next, you need to click on "Publish" to leave the edit and test mode and release the runbook.
6. [Create a scheduler](https://learn.microsoft.com/en-us/azure/automation/shared-resources/schedules#create-a-schedule) to run the script to update the WatchList automated on your preferred time interval.
7. Check the results of the WorkloadIdentityInfo table after a successful execution of the runbook.

    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect10.png)


## Enrichment of "Tiering Model" Classification

I’ve [published a definition of the Enterprise Access Model](https://github.com/Cloud-Architekt/AzurePrivilegedIAM) to classify and identify resources in Microsoft Entra ID. This classification model is part of my PoC project "EntraOps" and shared as community-driven project. Julian Hayward has implemented the EntraOps classification to [AzAdvertizer](https://www.azadvertizer.net/azEntraIdAPIpermissionsAdvertizer.html) and will be also available out-of-the-box in a future release of [AzADSPI](https://github.com/JulianHayward/AzADServicePrincipalInsights).


The classification files can be also used in KQL for enrichment and allows to identify App Roles and Directory Role assignment in relation to the definition of "Control Plane" privileges.
Therefore, I’ve created Microsoft Sentinel functions for AzADSPI and WorkloadIdentityInfo which allows to apply the classification (as external data from the JSON file on the GitHub project) by executing the KQL query.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect25.png){: width="80%" }

🔗 Function **"PrivilegedAzADSPI" for "AzADSPI" (Custom Table)**
[AzureSentinel/Functions/AzADSPI_EnrichedByEntraOps.yaml at main · Cloud-Architekt/AzureSentinel (github.com)](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Functions/AzADSPI_EnrichedByEntraOps.yaml)
🔗 Function **"PrivilegedWorkloadIdentity" for "WorkloadIdentityInfo" (WatchList)**
[AzureSentinel/Functions/PrivilegedIdentityInfo.yaml at main · Cloud-Architekt/AzureSentinel (github.com)](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Functions/PrivilegedWorkloadIdentityInfo.yaml)

### Classification of App Owner or Delegated Role Member

I’m planning to apply classification for all privileged users in Entra ID as part of the upcoming release of EntraOps which will cover all eligible, active and permanent members. In the meanwhile, I’m using the `IdentityInfo` table to get a list of all active or permanent Entra ID role member of the last 14 days. As already described before, EntraOps classification can be used to apply a general classification by adding them to the  KQL logic.

```powershell
// List of (active/permanent) Directory role member with with enriched classification from EntraOps Privileged EAM
// by using IdentityInfo table from Microsoft Sentinel UEBA
let SensitiveEntraDirectoryRoles = externaldata(RoleName: string, RoleId: string, isPrivileged: bool, Classification: dynamic)["https://raw.githubusercontent.com/Cloud-Architekt/AzurePrivilegedIAM/main/Classification/Classification_EntraIdDirectoryRoles.json"] with(format='multijson')
| where Classification.EAMTierLevelName != "Unclassified"
| extend EAMTierLevelName = Classification.EAMTierLevelName
| project RoleName, isPrivileged, EAMTierLevelName;
let SensitiveUsers = IdentityInfo
| where TimeGenerated > ago(14d)
| summarize arg_max(TimeGenerated, *) by AccountObjectId
| mv-expand AssignedRoles
| extend RoleName = tostring(AssignedRoles)
| join kind=inner ( SensitiveEntraDirectoryRoles ) on RoleName;
SensitiveUsers
| project EAMTierLevelName, RoleName, AccountObjectId, AccountDisplayName, AccountUPN, IsAccountEnabled, UserType, SourceSystem
```

In addition, I’ve created just another function which allows you to get a combined list of all identities (human and workload identities) which are available from the `IdentityInfo` and the custom solution `WorkloadIdentityInfo` . This includes also the classification enrichment.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect26.png)

🔗 Function "**UnifiedIdentityInfo**" for WatchList "WorkloadIdentityInfo" and UEBA table "IdentityInfo"
[AzureSentinel/Functions/UnifiedIdentityInfo.yaml at main · Cloud-Architekt/AzureSentinel (github.com)](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Functions/UnifiedIdentityInfo.yaml)

### **List of privileged assignments outside of Custom Table or WatchList**

Sometimes it’s required to get a list of the recent added privileged assignments directly from the `AuditLog` to cover also the latest assignments before the next automation schedule or trigger will update the Custom Table or WatchList. Therefore, I’ve created a function which shows all Microsoft Graph API and Entra ID role assignments of the latest 24 hours.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect11.png)

🔗 Function "**RecentAddedPrivileges"** to get latest MS Graph API and Entra ID role assignments from Audit Logs:
[AzureSentinel/Functions/RecentAddedPrivileges.yaml at main · Cloud-Architekt/AzureSentinel (github.com)](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Functions/RecentAddedPrivileges.yaml)

## Enriched Incidents from Microsoft Security products

In the following section, I like to give some examples how alerts from Microsoft Security products can be enriched by the WorkloadIdentityInfo WatchList.

### Risky Workload ID detected by Entra ID Protection

Leaked credentials of Workload Identity has been detected by Entra ID Protection. As already described in the previous part of the blog post series, the entity mapping is missing in Microsoft Sentinel. Therefore, I've written analytic rule which uses existing Alert (from the `SecurityAlert` table entry) to enrich them with detailed information from the WorkloadIdentityInfo watchlist in combination with the function `PrivilegedWorkloadIdentity`. This allows us to have an entity mapping to the Application object but also enriched information, including the Tiering Level of Privileged Access.

Below you’ll see the differences between the original incident on the left side and the enriched incident details (including privileged classification and entity details) from my custom analytics rules on the right side.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect12.png)

You will find the analytics rule templates on my repository:
🧪 [**Workload ID Protection Alerts with Enriched Information**](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Detections/EID-WorkloadIdentities/Workload%20ID%20Protection%20Alerts%20with%20Enriched%20Information.yaml)

_Side Note: Using this custom analytic rule could lead to duplicated incidents and alerts if incident creation has been already enabled for the alerts. Avoid to use incident creation rules for this type of Identity Protection Alerts or use an automation rule to avoid duplicates._

### Suspicious activities detected by MDA App Governance

MDA App Governance includes many built-in alert policies but also the option to configure customized patterns to detect suspicious activity. But similar to Entra ID Protection alerts for Workload Identities, the mapping to the entity and description details are also missing. Therefore, I’ve created an rule template to parse the `ApplicationId` from the "AlertLink URL" and correlate them with the `WorkloadIdentityInfo` for entity enrichment. In addition, the description details are not visible in the incident overview.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect13.png){: width="75%" }

As you can see in the next sample, a high volume of e-mail search activities has been detected by using a privileged interface. Information about the Workload Identity from the WatchList and classification by using the `PrivilegedWorkloadIdentityInfo` allows to add some related custom alert detail fields to the incident.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect13.png){: width="75%" }

You will find the analytics rule templates on my repository:
🧪 **[Workload ID Protection Alerts with Enriched Information](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Detections/EID-WorkloadIdentities/Workload%20ID%20Protection%20Alerts%20with%20Enriched%20Information.yaml)**

### Threat policy (anomaly detection) alerts from Microsoft Defender XDR

Microsoft Defender XDR includes a few [anomaly detection policies](https://learn.microsoft.com/en-us/defender-cloud-apps/anomaly-detection-policy#anomaly-detection-policies) for OAuth apps (e.g., [Unusual ISP for an OAuth App](https://learn.microsoft.com/en-us/defender-cloud-apps/anomaly-detection-policy#unusual-isp-for-an-oauth-app)). We are using just another analytics rules to use the entry from the `SecurityAlert` to create an enriched incident.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect15.png){: width="70%" }

The previous named templates can be found here:

🧪 **[MDA Threat detection policy for OAuth Apps with Enriched Information](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Detections/EID-WorkloadIdentities/MDA%20Threat%20detection%20policy%20for%20OAuth%20Apps%20with%20Enriched%20Information%20(WorkloadIdentityInfo).yaml)**

## Hunting queries with enriched data of Microsoft Security products

### Anomalous changes on high-sensitive Workload Identities

We have two interesting data sources which can be used for anomaly-based detection:
Microsoft Defender XDR Behaviors can be used for checking unusual addition of credentials, as you can see in the following hunting query. Permissions will be enriched with the data from the classification model.

```jsx
let SensitiveMsGraphPermissions = externaldata(EAMTierLevelName: string, Category: string, AppRoleDisplayName: string)["https://raw.githubusercontent.com/Cloud-Architekt/AzurePrivilegedIAM/main/Classification/Classification_AppRoles.json"] with(format='multijson');
BehaviorInfo
| where ActionType == "UnusualAdditionOfCredentialsToAnOauthApp"
| join BehaviorEntities on BehaviorId
| where EntityType == "OAuthApplication"
| extend Permissions = parse_json(AdditionalFields1).Permissions
| mv-expand Permissions
| extend Permission = tostring(Permissions.PermissionName)
| join kind=inner ( SensitiveMsGraphPermissions | project EnterpriseAccessModelTiering = EAMTierLevelName, EnterpriseAccessModelCategory = Category, AppRoleDisplayName ) on $left.Permission == $right.AppRoleDisplayName
| summarize ApplicationPermissions = make_list(Permission), EnterpriseAccessModelTiering = make_set(EnterpriseAccessModelTiering), EnterpriseAccessModelCategory = make_set(EnterpriseAccessModelCategory) by Timestamp, BehaviorId, ActionType, Description, Categories, AttackTechniques, ServiceSource, DetectionSource, DataSources, AccountUpn, Application, ApplicationId
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect16.png){: width="75%" }

The User Behavior Entity Analytics in Microsoft Sentinel includes also anomalies about "Application Management" activities from the user. This use case is not limited to a hunting query and would be also a potential anomaly-based detection. In this sample, we use the UEBA tables in combination with the `IdentityInfo` to build an analytics rule in Microsoft Sentinel for create an incident with "Medium" severity under the following conditions

- Investigation Priority Score is greater than 1 in combination if one of the conditions will be satisfied:
    - Actor has no active or permanent Entra ID role assignment in the past 14 days
    - Risky User with Risk Level of Medium or higher

The incident will be increased to "High" severity if the Actor has been assigned to "Control Plane" Entra ID role in the past 14 days. All other results will be set to severity "Informational" and not included in the incident creation.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect17.png)

You can find the KQL logic which is also part of the analytic rule-version of this hunting query:
🧪 [**UEBA Behavior anomaly on Application Management**](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Detections/EID-WorkloadIdentities/UEBA%20Behavior%20anomaly%20on%20Application%20Management.yaml)

### Hunting compromised owner of Application and Service principal object

As already described, AzADSPI includes some advanced details of the Workload Identities objects (such as Application or Service Principal Owner). This data is now available in Sentinel by ingesting the data to the custom table. Therefore, we can use them in a hunting query to find risk events related to identities which has also ownership of Workload Identities.

The query is very simple and just correlates the ownership details from the custom table with the  `AADRiskEvents` table from Entra ID Protection:

```jsx
let ServicePrincipalOwner = PrivilegedAzADSPI
    | mv-expand parse_json(ServicePrincipalOwners)
    | extend OwnerId = tostring(ServicePrincipalOwners.id)
    | extend Ownership = "ServicePrincipalObject";
let ApplicationOwner = PrivilegedAzADSPI
    | mv-expand parse_json(ApplicationOwners)
    | extend OwnerId = tostring(ApplicationOwners.id)
    | extend Ownership = "ApplicationObject";
AADUserRiskEvents
| where TimeGenerated >ago(365d)
| join kind=inner (
    union ServicePrincipalOwner, ApplicationOwner
) on $left.UserId == $right.OwnerId
| project TimeGenerated, UserDisplayName, UserId, UserPrincipalName, RiskEventType, RiskDetail, RiskLevel, RiskState, WorkloadIdentityName, WorkloadIdentityType, Ownership
```

The result can be look like this one, attempt to get access to a Primary Refresh Token of a user has been detected which has access to a Workload Identity. Optionally, the classification of the affected workload identity could be also added.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect18.png)

### Attack path from unsecured workload environments

KQL offers the option to create "cross-service queries" which give us the chance to get results from the Azure Resource Graph in combination with data in the Microsoft Sentinel Workspace. At time of writing the blog post, this is not supported for Analytics Rules. However, it can be used in hunting queries. In this sample, we will use this feature to get a list of all attack paths from "Microsoft Defender for Cloud" CSPM feature which includes a Workload Identity as entity. Afterwards we use the result to enrich them with data from the WorkloadIdentityInfo table.

```jsx
arg("").securityresources
    | where type == "microsoft.security/attackpaths"
    | extend AttackPathDisplayName = tostring(properties["displayName"])
    | mvexpand (properties.graphComponent.entities)
    | extend Entity = parse_json(properties_graphComponent_entities)
    | extend EntityType = (Entity.entityType)
    | extend EntityName = (Entity.entityName)
    | extend EntityResourceId = (Entity.entityIdentifiers.azureResourceId)
    | where EntityType == "serviceprincipal" or EntityType == "managedidentity"
    | project id, AttackPathDisplayName, tostring(EntityName), EntityType, Description = tostring(properties["description"]), RiskFactors = tostring(properties["riskFactors"]), MitreTtp = tostring(properties["mITRETacticsAndTechniques"]), AttackStory = tostring(properties["attackStory"]), RiskLevel = tostring(properties["riskLevel"]), Target = tostring(properties["target"])
| join hint.remote=right ( PrivilegedWorkloadIdentityInfo
) on $left.EntityName == $right.ServicePrincipalObjectId
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect19.png)

In the result, we can see the details from the MDC Attack Path but also the assigned permissions and classification of the affected workload identity.

*Side Note: You can also build own attack correlation with the data from AzADSPI. The ingested data includes the columns `ManagedIdentityAssociatedAzureResources` and `ManagedIdentityFederatedIdentityCredentials` which gives you the chance to correlate the Workload Identity to the assigned Azure or Federated Entity (such as GitHub or Kubernetes workload).*

## Custom analytics rules with WorkloadIdentityInfo in Microsoft Sentinel

### Added Ownership on privileged workload identity to lower-privileged user

In general, assignment of owners to Application and Service Principals should be avoided. Microsoft offers already a rule template ("[Add owner to application](https://github.com/Azure/Azure-Sentinel/blob/3141233390c9f731c7c9772f76e70f91173e9688/Detections/AuditLogs/ChangestoApplicationOwnership.yaml#L4)") to cover this use case for Application object. But keep in mind, also an owner of service principal object is able to issue a valid credential to the workload identity.

I’ve created an analytics rule which will generate an incident with high severity if an ownership of Application or Service Principal has been assigned to a user which has lower privileges than the application. Not only the actor will be included in the incident, also the target principal is part of the entity mapping.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect20.png)

This rule template can be found here and can be used in combination with the WorkloadIdentityInfo:

**🧪[Added Ownership to workload identity (WorkloadIdentityInfo)](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Detections/EID-WorkloadIdentities/Added%20Ownership%20to%20workload%20identity%20(WorkloadIdentityInfo).yaml)**

### Added credential on privileged workload identity by lower privileged users

Issuing a credential by an owner is a sensitive management operation. Especially, if the owner has the chance use an impersonation of the workload identity for privilege escalation. In the next example, I’m using the classification of the workload but also actor to identify if the credential has been issued by a lower privileged user. This incident can be also correlate to a previous incident ("Added Ownership on privileged workload identity to lower-privileged user") to understand the context of both security events as multi-stage attack.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect21.png)

You can find the analytic rule in my repository and should be deployed in combination with the previous rule template:

**🧪 [Added Credential to privileged workload by lower or non-privileged user](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Detections/EID-WorkloadIdentities/Added%20Credential%20to%20privileged%20workload%20by%20lower%20or%20non-privileged%20user%20(WorkloadIdentityInfo).yaml)**

### Token Replay from compromised or unsecured workload environments

I’ve build and shared some hunting queries for correlation between a sign-in and activity event from the `AzureActivity` and the new `MicrosoftGraphActivityLogs` in [October this year](https://twitter.com/Thomas_Live/status/1713812016484450589). This can be used as potential indicator for token replay in my opinion.

Replay of tokens from unsecured DevOps or other workload environments has been one of my live demos in previous community talks about workload identities. Correlation between IP address from sign-in and activity was of the few available options before (e.g., [Azure Activity with Federated Credentials outside of GitHub Workflow activity](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Detections/GitHub/FedCredAzActivityWithoutSignInWorkflow.yaml)). But now we can build additional queries to use the `UniqueTokenIdentifier` to build a correlation between the acquired token from the sign-in process and the activity of the issued token.

Dynamic severity can be set on conditions if the IP address is suspicious or is coming from unfamiliar IP service tags/location. In this sample, the severity is increased to high if it’s outside of Azure.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-12-18-workload-id-advanced-detection-enrichment/workloadidadvdetect22.png){: width="45%" }

The analytic rule logic is available for Azure Resource Manager and Microsoft Graph here:

**🧪 [Token Replay from workload identity with privileges in Microsoft Azure (WorkloadIdentityInfo).yaml](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Detections/EID-WorkloadIdentities/Token%20Replay%20from%20workload%20identity%20with%20privileges%20in%20Microsoft%20Azure%20(WorkloadIdentityInfo).yaml)**

**🧪 [Token Replay from workload identity with privileges in Microsoft Entra or Microsoft 365 (WorkloadIdentityInfo)](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Detections/EID-WorkloadIdentities/Token%20Replay%20from%20workload%20identity%20with%20privileges%20in%20Microsoft%20365%20(WorkloadIdentityInfo).yaml)**

## Next: Incident Response and Conditional Access

In the next part of the blog post series, we will go into details about incident response on Workload Identities and how Conditional Access can be used for automated response. So, stay tuned!