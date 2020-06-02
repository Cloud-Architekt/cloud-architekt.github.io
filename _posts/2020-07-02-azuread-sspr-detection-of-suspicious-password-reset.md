---
layout: post
title:  "Azure AD SSPR: Detection of suspicious password resets"
author: thomas
categories: [ Azure, Security, AzureAD ]
tags: [security, azuread]
image: assets/images/sspr.png
description: "End-users are able to reset their passwords as part of the Azure AD „self-service password reset“ (SSPR) service. Including an option to write back passwords resets from Azure AD to on-premises AD. Consideration of security aspects and detection of any suspicious activity in the password reset process should be included in your implementation."
featured: false
hidden: false
---

## SSPR auditing and insights
### Azure AD Audit Logs
All steps of the SSPR user flow is audited by (service) source „Self-service password reset“ as part of the Azure AD Audit Log.

The following screenshots shows the record of a successfully reset by a cloud-only identity in Azure AD audit blade:

![](../2020-07-02-azuread-sspr-deployment-considerations/ssprauditcloudonly.png)

As you can see the reset was initiated by fim_password_service@support.onmicrosoft.com. This is an internal account that will be used for resetting the password in the Core Directory (as part of an authorized/valid SSPR request). All other events will be recorded as initiated by the username which was entered as part of the SSPR user flow.

A detailed view on the logs (from Log Analytics) shows that all SSPR-related events shares the same Correlation Id.
![](../2020-07-02-azuread-sspr-deployment-considerations/sspraudituserflow.png)

#### Hybrid identity environments
Users in a hybrid environment (and enabled password write-back) will be facing a very similar set of audit events. In addition you can see the activities by the “Sync_<AADConnectServerName>” account which initiate the password change from an Azure AD Connect Server to Active Directory.

![](../2020-07-02-azuread-sspr-deployment-considerations/sspraudithybrid.png)

CorrelationId of Azure AD SSPR events is also part of the local AAD Connect Eventlog even if the “fim_password_service” and “Sync_*” initiated events have different CorrelationIds.

### Azure Sentinel Workbook
Visualized overview about numbers of operations (including SuccessRate) of password resets by admins or user (self-service) is available in Azure Sentinel. It’s part of the workbook “Azure AD Audit, Activity and Sign-in logs” and based on KQL queries of Azure AD audit logs by Sentinel’s data connector.

![](../2020-07-02-azuread-sspr-deployment-considerations/ssprsentinel.jpg)

### Azure AD Usage and insights
An overview of registered authentication methods by users are available in the “[Usage & Insights](https://docs.microsoft.com/en-us/azure/active-directory/authentication/howto-authentication-methods-usage-insights)” dashboard. 

![](../2020-07-02-azuread-sspr-deployment-considerations/sspraadinsights.jpg)

### Microsoft Cloud App Security (MCAS) Activity Log
Activity log in MCAS gives you an overview about all password changes as part of Azure AD SSPR, change password in Azure AD and on-premises AD. You are able to get the list by filtering the Activity Query to"Password changes and reset requests":

![](../2020-07-02-azuread-sspr-deployment-considerations/ssprmcasfilter.jpg)

![](../2020-07-02-azuread-sspr-deployment-considerations/ssprmcasoverview.jpg)

SSPR attempts and detailed steps from the user flow in SSPR are not included in this activity log. You get a general event entry in case of successful requests. 

![](../2020-07-02-azuread-sspr-deployment-considerations/ssprmcas.png)


## Detect suspicious password reset by users and admins with Azure Sentinel
At the moment there is no detection for suspicious activities of SSPR request available. Therefore I started to write some KQL queries that can be used as part of an Azure Sentinel Analytic rules to create incidents in such cases. You‘ll find the latest version of the queries on my GitHub repository. It’s free use but without given any guarantee or support: https://github.com/Cloud-Architekt/azuresentinel

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

Thanks to Marcel Meurer for supporting me in writing the correlation between Sign-in and AuditLog data in KQL.

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
The next query is very simple but effective. Alert will be triggered if an admin have reset the password and modification of StrongAuthenticationMethod was stated. Maximum time difference between both events can be adjusted. This could be an indication of account takeover as part of suspicious activity.

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




