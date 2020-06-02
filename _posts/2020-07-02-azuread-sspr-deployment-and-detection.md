---
layout: post
title:  "Azure AD SSPR: Deployment considerations and detection of suspicious self-service password reset"
author: thomas
categories: [ Azure, Security, AzureAD ]
tags: [security, azuread]
image: assets/images/sspr.png
description: "End-users are able to reset their passwords as part of the Azure AD „self-service password reset“ (SSPR) service. Including an option to write back passwords resets from Azure AD to on-premises AD. Consideration of security aspects and detection of any suspicious activity in the password reset process should be included in your implementation."
featured: false
hidden: false
---

_End-users are able to reset their passwords as part of the Azure AD „self-service password reset“ (SSPR) service. Including an option of password writeback from Azure AD to on-premises AD. Consideration of security aspects and detection of any suspicious activity in the password reset process should be included in your deployment plan._

## First off: Relevance of password resets in an era of “passwordless”
Most organizations are moving to passwordless options to “kill” passwords and all the weakness and (risk) management that comes with it.
There is a wide range of options such as Windows Hello, Microsoft Authenticator and FIDO2.

Nevertheless, currently there are use cases where initial user (self-service) enrollments depends on password-based credentials or fallback options are required (e.g. to enable or recover passwordless authenticators).
In addition there’s no option to force “passwordless only” yet. 
I guess many of those “challenges” will be resolved or fixed in the near feature.

If, however, passwords will be used in rare cases it’s more likely that users will forget their credentials.  Many organizations have been chosen similar authentication methods for reset passwords and 2nd factor authentication. This could be an interesting attack scenario to gain (strong authentication) credentials if it’s been even harder to attack passwordless authentication methods.

## Overview and Architecture
At first, it is necessary to know the architecture and technical flow before we go into details about design considerations and options for securing and monitoring SSPR events in Azure AD.

So check out Microsoft’s description of [Key benefits](https://docs.microsoft.com/en-us/azure/active-directory/authentication/howto-sspr-deployment#key-benefits) and [User flow](https://docs.microsoft.com/en-us/azure/active-directory/authentication/howto-sspr-deployment#solution-architecture) if you aren‘t already familiar with the fundamentals of Azure AD SSPR. This content is part of [planning guide of SSPR deployment](https://docs.microsoft.com/en-us/azure/active-directory/authentication/howto-sspr-deployment#plan-configuration) which includes many guidance and best practices by Microsoft. In my opinion it’s a great start to get an good overview as you can see in this overview graphic of the user flow:
 
![](../2020-07-02-azuread-sspr-deployment-detection/sspruserflow.png)

I can also strongly recommended to watch the [official learning videos about "Azure AD SSPR" on YouTube](https://www.youtube.com/watch?v=hc97Yx5PJiM&feature=youtu.be).

## Considerations in Deployment and User Flow
### Access to password reset page
User flow within the SSPR portal is very straight forward and [well documented by Microsoft](https://docs.microsoft.com/en-us/azure/active-directory/authentication/concept-sspr-howitworks#how-does-the-password-reset-portal-work) (including prerequisite checks to verify that users are able to reset their passwords).

In the most scenarios the users will be forwarded to SSPR by using the link „Can‘t access your account?“ or “Forgot my password” at the organization‘s Azure AD sign-in page. But there is also an short URL link available (https://aka.ms/sspr).

![](../2020-07-02-azuread-sspr-deployment-detection/ssprgate.jpg)

The option to start the SSPR process is public available for everyone who is able to enter the UserID and CAPTCHA. This has been implemented by Microsoft to prevent bots from using the SSPR gate.

Unfortunately the initial access to this portal can not be restricted or protected by any further advanced options. 

_Note: In my opinion it would be helpful to give admins the option to restrict access by conditions such as trusted location or registered devices. It seems there are no built-in security options which also prevents access from risky or anonymous IP address._

An overview of all authentication methods of the users is visible as part of the next step:

![](../2020-07-02-azuread-sspr-deployment-detection/ssprgateauthmethods.png)

Details of the phone number or mail addresses will be hidden. It is, however, possible to see the domain name and first characters of the alternate mail addresses.

Failed attempts of the CAPTCHA input will not be audited but all other events are part of the Azure AD Audit Log:

![](../2020-07-02-azuread-sspr-deployment-detection/ssprgatelogscaptcha.jpg)

More details on auditing of SSPR will be explained later in this blog post.

### Throttling of multiple attempts to reset passwords
Microsoft has implemented some limitations and locked out processes to reduce number of SSPR attempts:

* Users can try only five password reset attempts within a 24 hour period before they're locked out for 24 hours.
	
* Users can try to validate a phone number, send a SMS, or validate security questions and answers only five times within an hour before they're locked out for 24 hours.
	
* Users can send an email a maximum of 10 times within a 10 minute period before they're locked out for 24 hours.
	
* The counters are reset once a user resets their password.

_Source: [Password management frequently asked questions](https://docs.microsoft.com/en-us/azure/active-directory/authentication/active-directory-passwords-faq#password-reset)_

### Write-back to Active Directory (on-premises)
SSPR gives you also the option to enable password writeback to on-premises / Active Directory. An detailed tutorial to [configure and enable the write-back option](https://docs.microsoft.com/en-us/azure/active-directory/authentication/tutorial-enable-sspr-writeback) is available on Microsoft Docs.

All events of the SSPR write-back will be audited in the Application Eventlog (Source „PasswordResetService“) on the Azure AD Connect Servers:

![](../2020-07-02-azuread-sspr-deployment-detection/aadconnecteventvwr.png)

EventLog entries includes a „TrackingId“ which is identical with the „CorrelationID“ in the Azure AD Audit Log. This allows to build an correlation between SSPR events in Azure AD and Active Directory.

![](../2020-07-02-azuread-sspr-deployment-detection/sspraadauditlog.jpg)

I can strongly recommended to delegate “write” permissions of passwords in AD to the certain scope of hybrid users which are planned to use SSPR.
[Aaron Guilmette](https://www.undocumented-features.com/about/) has written a [PowerShell script to set this permissions](https://www.powershellgallery.com/packages/AADConnectPermissions/) on scope of an organizational unit. 

_Note: There are reports in Microsoft (Support) forums where customers are running into issues in environments with implemented ATA and (domain-wide) configured SAM-R (for lateral movement). Check [this forum post](https://social.msdn.microsoft.com/Forums/SECURITY/en-US/6082daf5-2893-407b-b009-bc49464df984/aadsync-password-reset?forum=WindowsAzureAD) for further details._ 

### Other considerations
Some other general considerations and implementation questions around SSPR are already written down by Microsoft:

* SSPR requests of [B2B users (guests)](https://docs.microsoft.com/en-us/azure/active-directory/authentication/concept-sspr-howitworks#password-reset-for-b2b-users)
* "Password writeback" is a feature of Azure AD Connect, so keep the components up-to-date (no support for versions that were released more than 18 months).  Regular check of [version history](https://docs.microsoft.com/en-us/azure/active-directory/hybrid/reference-connect-health-version-history) is recommended to follow changes.
* [Frequently asked questions of SSPR management](https://docs.microsoft.com/en-us/azure/active-directory/authentication/active-directory-passwords-faq)
	(incl. password policy and cloud-only scenarios)

I can strongly recommend to have in-place an actively monitor of your SSPR audit events and insights (as described in the last section of this article).

## Registration and options of authentication methods methods for SSPR
Users must already have been registered for SSPR with authentication method(s) before they begin the reset process. It’s recommended to [enable combined registration for SSPR and MFA](https://docs.microsoft.com/en-us/azure/active-directory/authentication/concept-registration-mfa-sspr-combined) for those users. This step should be part of the (Azure AD) user on-boarding process and configured right from the start.

_Advice: It‘s important to restrict access of MFA and SSPR registration based on trusted location or device. Check out to [secure the registration process by Conditional Access](https://docs.microsoft.com/en-us/azure/active-directory/conditional-access/howto-conditional-access-policy-registration)._

But you should also verify the number of days where users will be prompted to re-confirm their authentication information:

![](../2020-07-02-azuread-sspr-deployment-detection/ssprregistration.png)

In General you should decide if you like to enable users to manage their authentication methods themselves or to manage them as part of IAM processes. I guess the most regulated or larger companies will facing some compliance and governance requirements around this decision.

_Tip: Microsoft introduced recently a [new Microsoft Graph (beta) API to manage users‘ authentication methods (combined registration) programmatically.](https://techcommunity.microsoft.com/t5/azure-active-directory-identity/manage-your-authentication-phone-numbers-and-more-in-new/ba-p/1257359). A highly-requested feature to pre-register and manage all aspects of Authenticator that are used for MFA and SSPR._ 

Alongside to configure the registration of SSPR as part of Conditional Access Policies you need to enable this feature and set a scope in the SSPR settings.

![](../2020-07-02-azuread-sspr-deployment-detection/ssprproperties.jpg)

_Note: Unfortunately there‘s no option to create multiple groups or configurations for SSPR. But there‘s already an [Azure Feedback](https://feedback.azure.com/forums/169401-azure-active-directory/suggestions/31990900-allow-multiple-groups-for-sspr-rather-than-only-on) for requesting multiple group support and the product group „planned“ to add some solution of this missing function._

### Options of Authentication methods
I can only strongly recommend to require two different methods for a SSPR request.

In my opinion it isn‘t easy to choose the right methods and you should try to find the right balance between security and dependency in case of the reset process:

* Authentication methods which can be configured by users to (personal) mobile device, phone number or mail address are possibly in access without PIN (stolen device scenario).
* Office phone could be a bad option if you are using softphones or Microsoft Teams only.
* Security questions have some disadvantages in usability (hard to remember the answers) or vulnerable for social engineering (if the questions and answers are too easy). 

Some of you will say... that‘s the reasons why two authentication methods are required. But it‘s even harder to find two good options.

You‘ll find a list of [supported authentication methods](https://docs.microsoft.com/en-us/azure/active-directory/authentication/concept-authentication-methods) and description in Microsoft Docs.

_Note: Admins have the option to enable SSPR requests from Windows login screens. It would be a great option if Microsoft allows to limit SSPR reset from user‘s owned device (in combination with registered or AAD-joined devices). This would helps to reduce the attack surface and can be include as additional authentication method._

## Notification of users
Currently it‘s limited to enable notification for admins and users via e-mail notification (as part of the SSPR configuration).

If you need any further options you should verify to build an automation with Azure Logic Apps or PowerAutomate.

_Note: Microsoft already [supports to sending security notifications](https://techcommunity.microsoft.com/t5/azure-active-directory-identity/microsoft-authenticator-app-now-sends-security-notifications/ba-p/328281) (such as event of a password change) for Microsoft Accounts as part of the Microsoft Authenticator App. I hope this feature will be added for Work accounts (Azure AD) as well._


## Unlocking accounts by SSPR
Active Directory accounts (on-premises) will also be „unlocked“ if user resets their password and writeback is enabled.

But you have also the option to separate both operations by enabling to „unlock accounts“ without resetting the password of the account:

![](../2020-07-02-azuread-sspr-deployment-detection/ssprwriteback.jpg)
 
## Other options to reset password of users
By default the following Directory Roles in Azure AD are able to reset passwords as "admin":
* Authenticator Administrator
* Helpdesk Administrator
* Password Administrator 
* Privileged Authentication Administrator
* User Administrator 
	
There’s no built-in option to notify users if admins are resetting their credentials. This activity is only covered by the Azure AD reports.

_Advice: Be aware of the various directory roles and their permission to reset passwords and „non-password credentials“ (authentication methods such as MFA or passwordless). A very good example is the „Authentication Administrator“ role which can reset passwords and other authentication methods of non-admins. This means assigned role members are able to „take over“ any „non-admin account“. In this case „non-admin“ also affect accounts with privileged permissions as subscription owner or outside of Azure AD RBAC (such as Intune or Exchange Online RBAC). In a case like this, it is always a good idea to check the detailed description and notes from Microsoft‘s documentation on „[Administrator role permissions](https://docs.microsoft.com/en-us/azure/active-directory/users-groups-roles/directory-assign-admin-roles#authentication-administrator)“._


## SSPR auditing and insights
### Azure AD Audit Logs
All steps of the SSPR user flow is audited by (service) source „Self-service password reset“ as part of the Azure AD Audit Log.

The following screenshots shows the record of a successfully reset by a cloud-only identity in Azure AD audit blade:

![](../2020-07-02-azuread-sspr-deployment-detection/ssprauditcloudonly.png)

As you can see the reset was initiated by fim_password_service@support.onmicrosoft.com. This is an internal account that will be used for resetting the password in the Core Directory (as part of an authorized/valid SSPR request). All other events will be recorded as initiated by the username which was entered as part of the SSPR user flow.

A detailed view on the logs (from Log Analytics) shows that all SSPR-related events shares the same Correlation Id.
![](../2020-07-02-azuread-sspr-deployment-detection/sspraudituserflow.png)

#### Hybrid identity environments
Users in a hybrid environment (and enabled password write-back) will be facing a very similar set of audit events. In addition you can see the activities by the “Sync_<AADConnectServerName>” account which initiate the password change from an Azure AD Connect Server to Active Directory.

![](../2020-07-02-azuread-sspr-deployment-detection/sspraudithybrid.png)

CorrelationId of Azure AD SSPR events is also part of the local AAD Connect Eventlog even if the “fim_password_service” and “Sync_*” initiated events have different CorrelationIds.

### Azure Sentinel Workbook
Visualized overview about numbers of operations (including SuccessRate) of password resets by admins or user (self-service) is available in Azure Sentinel. It’s part of the workbook “Azure AD Audit, Activity and Sign-in logs” and based on KQL queries of Azure AD audit logs by Sentinel’s data connector.

![](../2020-07-02-azuread-sspr-deployment-detection/ssprsentinel.jpg)

### Azure AD Usage and insights
An overview of registered authentication methods by users are available in the “[Usage & Insights](https://docs.microsoft.com/en-us/azure/active-directory/authentication/howto-authentication-methods-usage-insights)” dashboard. 

![](../2020-07-02-azuread-sspr-deployment-detection/sspraadinsights.jpg)

### Microsoft Cloud App Security (MCAS) Activity Log
Activity log in MCAS gives you an overview about all password changes as part of Azure AD SSPR, change password in Azure AD and on-premises AD. You are able to get the list by filtering the Activity Query to"Password changes and reset requests":

![](../2020-07-02-azuread-sspr-deployment-detection/ssprmcasfilter.jpg)

![](../2020-07-02-azuread-sspr-deployment-detection/ssprmcasoverview.jpg)

SSPR attempts and detailed steps from the user flow in SSPR are not included in this activity log. You get a general event entry in case of successful requests. 

![](../2020-07-02-azuread-sspr-deployment-detection/ssprmcas.png)


## Detect suspicious password reset by users and admins with Azure Sentinel
At the moment there is no detection for suspicious activities of SSPR request available (as far as I know). Therefore I started to write some KQL queries that can be used as part of an Azure Sentinel Analytic rules to create incidents in such cases. You‘ll find the latest version of the queries on my GitHub repository. It’s free use but without given any guarantee or support: [Cloud-Architekt/azuresentinel](https://github.com/Cloud-Architekt/azuresentinel)

### Blocked attempts of SSPR by user
This simple query is checking the AuditLog for SSPR events of „blocked“ attempts after reaching the throttling limit (within a specific time range).
This could be the case if an attacker is trying to guessing answers of the security questions.

```
let timeRange = 1d;

AuditLogs
| where TimeGenerated >= ago(timeRange)
| where LoggedByService == "Self-service Password Management" 
| where OperationName contains "Blocked"
| extend UserPrincipalName = tostring(TargetResources[0].userPrincipalName), SourceIPAddress = tostring(InitiatedBy.user.ipAddress)
| project timestamp = TimeGenerated, AccountCustomEntity = UserPrincipalName, IPCustomEntity = SourceIPAddress, OperationName, ResultReason

```

### Suspicious activity from unknown or risky IP address to SSPR
The following query is far more complex. All SSPR user flows will be correlated with sign-in events of the user. The result includes all attempts from an IP address that was not used for successfully logons or detected as risky sign-in (within a previous timeframe).

You are able to define the time between SSPR request and prior sign-in by „maxTimeBetweenSSPRandSigninInMinutes“. It makes sense to adjust this value based on your environment and user behavior.

Thanks to [Marcel Meurer](https://twitter.com/marcelmeurer) for supporting me in writing the correlation between Sign-in and AuditLog data in KQL.

```
let timeRange = 1d;
let maxTimeBetweenSSPRandSigninInMinutes=7*24*60; // per Default max. difference is set to 7 Days

 
AuditLogs
| where TimeGenerated >= ago(timeRange) 
| where LoggedByService == "Self-service Password Management" and ResultDescription == "User submitted their user ID"
| extend AccountType = tostring(TargetResources[0].type), UserPrincipalName = tostring(TargetResources[0].userPrincipalName), TargetResourceName = tolower(tostring(TargetResources[0].displayName)), SSPRSourceIP = tostring(InitiatedBy.user.ipAddress)

| project UserPrincipalName, SSPRSourceIP, SSPRAttemptTime = TimeGenerated, CorrelationId

| join kind= leftouter (
   SigninLogs
   | where datetime_add('minute',maxTimeBetweenSSPRandSigninInMinutes,TimeGenerated) >= ago(timeRange)
   | where ResultType == "0"
   | where RiskLevelAggregated == "none" or RiskLevelDuringSignIn == "none"
   | extend TrustedIP = tostring(IPAddress)
   | project UserPrincipalName, TrustedIP, SignInTime = TimeGenerated
) on UserPrincipalName

| where SSPRAttemptTime > SignInTime
| extend TimeDifferenceInMinutes= iif(SSPRSourceIP==TrustedIP,datetime_diff("Minute",SignInTime,SSPRAttemptTime), 0), Match=SSPRSourceIP==TrustedIP
| where TimeDifferenceInMinutes >= -maxTimeBetweenSSPRandSigninInMinutes
| summarize  SignInsFromTheSameIP=countif(Match), min(TimeDifferenceInMinutes) by UserPrincipalName, CorrelationId, SSPRAttemptTime, SSPRSourceIP   //SignInsFromTheSameIP=0 if no sign in came from the IP used for SSPR in the last maxTimeBetweenSSPRandSigninInMinutes
| where SignInsFromTheSameIP == "0"
| project timestamp = SSPRAttemptTime, AccountCustomEntity = UserPrincipalName, IPCustomEntity = SSPRSourceIP
```

### Reset of two-factor authentication credentials by admin
The next query is very simple but effective. Alert will be triggered if an admin have reset the password and modification of StrongAuthenticationMethod was stated. Maximum time difference between both events can be adjusted. This could be an indication of account takeover by a compromissed account.

```
let timeFrame = 1h;
let resetDiff = 10m;

  AuditLogs
 | where TimeGenerated >= ago(timeFrame) 
 | where OperationName == "Reset password (by admin)" 
 | extend PasswordResetTime = TimeGenerated, UserPrincipalName = tostring(TargetResources[0].userPrincipalName), PasswordResetIP = tostring(InitiatedBy.user.ipAddress)

  | join kind= inner (

      AuditLogs
      | where TimeGenerated >= ago(timeFrame) 
      | where TargetResources contains "StrongAuthenticationMethod"
      | extend StrongAuthModifyTime = TimeGenerated, UserPrincipalName = tostring(TargetResources[0].userPrincipalName)

    // Audit Event contains no source IP, using OperationsName "Admin updated security info" is not covering reset of MFA
 ) on UserPrincipalName 
  | where PasswordResetTime - StrongAuthModifyTime <= resetDiff or StrongAuthModifyTime - PasswordResetTime <= resetDiff
  | summarize PasswordResetTime = max(PasswordResetTime), StrongAuthModifyTime = max(StrongAuthModifyTime) by UserPrincipalName, PasswordResetIP 
  | extend timestamp = PasswordResetTime, AccountCustomEntity = UserPrincipalName, IPCustomEntity = PasswordResetIP
```






<br>
<br>
<br>


<span style="color:silver;font-style:italic;font-size:small">Original cover image by [mohamed Hassan / Pixabay](https://pixabay.com/illustrations/cyber-security-security-lock-3216076/)</span>



