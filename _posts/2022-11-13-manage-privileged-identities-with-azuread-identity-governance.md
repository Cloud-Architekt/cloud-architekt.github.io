---
title: "Manage user lifecycle of Privileged Identities with Azure AD Identity Governance"
excerpt: "Microsoft has been released a feature to automate on- and off-boarding tasks for Azure AD accounts. Lifecycle workflows offers built-in workflow templates but also the option to integrate Logic Apps as custom extensions. In this blog post, I like to show, how to use this feature to automate the lifecycle of privileged accounts in association with a hiring and termination process"
header:
  overlay_image: /assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow1.png
  overlay_filter: rgba(102, 102, 153, 0.85)
  teaser: /assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow1.png
search: true
toc: true
toc_sticky: true
categories:
  - Azure AD
tags:
  - AzureAD
  - PrivilegedIAM
  - IdentityGovernance 
last_modified_at: 2022-11-14
---

*Microsoft has been released a feature to automate on- and off-boarding tasks for Azure AD accounts. Lifecycle workflows offers built-in workflow templates but also the option to integrate Logic Apps as custom extensions. In this blog post, I like to show, how to use this feature to automate the lifecycle of privileged accounts in association with a hiring and termination process.*

*This article is part of an upcoming blog series about “Securing Privileged Access”…*
<br>

## Foundation of Privileged Accounts

Microsoft recommends using cloud-only and dedicated user accounts for privileged access. They should be mastered in Azure Active Directory (without synchronization or dependency from Active Directory) to isolate those accounts in the case of an on-premises compromise. In the past, it was a challenge to manage the lifecycle of those accounts. Custom scripts, 3rd party solutions or manual processes have been implemented for (de)provisioning of privileged accounts. Built-in capabilities of [Cloud HR-driven user provisioning](https://learn.microsoft.com/en-us/azure/active-directory/app-provisioning/what-is-hr-driven-provisioning) needs to have supported systems in place (e.g. WorkDay or SAP SuccessFactor).

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/M365AdminIsolation.png)

*Overview of isolated Azure AD and Microsoft 365 administration from Microsoft Docs article "[Protecting Microsoft 365 from on-premises attacks](https://learn.microsoft.com/en-us/azure/active-directory/fundamentals/protect-m365-from-on-premises-attacks)”. TL;DR: “No on-premises accounts should have administrative privileges in Microsoft 365.”* 

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow1.png)

### Why should you isolate between work and privileged account?

In my opinion, separation between work account (for productivity tasks) and privileged accounts (for managing infrastructure or workloads) are necessary to enforce strong security and access policies but also a particular monitoring. For example, enforcing usage of Privileged Admin Workstations (PAW) by Conditional Access Policies or limited Internet/mailbox access cannot be achieved if you are using a single user account. In my opinion, the risk of identity compromise on work account is higher because increased attack surface by everyday access to various productivity workloads and access to public internet. Separation between regular user and privileged access is a key foundation if you want to implement an [Enterprise Access Model](https://aka.ms/SPA).

Additional references to Microsoft’s recommendations:

- [Secure access practices for administrators in Azure AD - Microsoft Entra | Microsoft Learn](https://learn.microsoft.com/en-us/azure/active-directory/roles/security-planning#ensure-separate-user-accounts-and-mail-forwarding-for-global-administrator-accounts)
- [Step 2. Protect your Microsoft 365 privileged accounts - Microsoft 365 Enterprise | Microsoft Learn](https://learn.microsoft.com/en-us/microsoft-365/enterprise/protect-your-global-administrator-accounts?view=o365-worldwide)

### Definition and requirements of Azure AD privileged accounts

So, let’s summarize what are the key aspects and requirements for the user type of privileged identities in my following use case:

- Creation of cloud-only account which are separated from “work account”
- Assignment of required licenses (Azure AD P2) only and avoid assigning licenses/access to productive workloads (Microsoft Teams)
- Option to create separated and multiple privileged accounts for managing different environment (B2C tenant, staging environments) or access levels (based on Tiering/Enterprise Access Model, e.g. Control and Management Plane)
- Relation between work and privileged account to trigger lifecycle events (joiner, leaver) and build correlation for monitoring
- No assigned user mailbox, notifications will be redirected/forwarded to work account
- Account will be created without pre-provisioned privileges, membership or access packages. Assignments needs to be requested by new privileged user (and approved as part of Entitlement Management)
- “Passwordless” onboarding by using “[Temporary Access Pass](https://learn.microsoft.com/en-us/azure/active-directory/authentication/howto-authentication-temporary-access-pass)” (very time-limited) to configure FIDO2 security key or Windows Hello for Business (WHfB) on admin workstations (PAW/SAW).

## Identity Lifecycle Workflows

I’ve chosen the public preview of the new “[Lifecycle Workflow](https://learn.microsoft.com/en-us/azure/active-directory/governance/what-are-lifecycle-workflows)” feature from (“Microsoft Entra Identity Governance”) to automate the on- and offboarding process for privileged accounts. This allows to use information from the source of authority (HR system but also Azure AD or Active Directory) to trigger the process and get required attributes for (de)provisioning of the privileged account.

Even if you are using Active Directory as source of authority to trigger (de)provisioning of privileged accounts, a comprehensive isolation is given (in my opinion). Only the trigger event to create and disable a user account relies on the Active Directory. From my point of view, an abuse and impact are very limited in case of a compromise. 

*Side Note: Even I’m using the terminology or object name “privileged users”, those accounts are not assigned to privileged roles or permissions at the time of onboarding. Afterwards, a separated process for (Privileged) Entitlement Management will be responsible to verify and assign those privileges.*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow2.png)

*Lifecycle Management in Microsoft Entra is the heart of Azure AD to automate the provisioning of users. Lifecyle workflows enables you to implement built-in templates for on- and offboarding but also create custom workflows for integration of advanced scenarios. Logic Apps are the platform to create custom extensions. Image Source: [User Lifecycle Management Software | Microsoft Security](https://www.microsoft.com/en-us/security/business/identity-access/azure-active-directory-lifecycle-management-software).*

### Trigger the provisioning process in relation to work account

Lifecycle workflows will be triggered based on the attributes `EmployeeHireDate`  and `EmployeeLeaveDate`  in Azure AD. These attributes from the work account of the employee will be used to trigger the event for (de)provisioning of privileged accounts synchronously to the hiring and termination process. It’s required that this fields will be synchronized from the source of authority. For the following sample of use, I’ve considered the following the options:

**Cloud-only work accounts: Provisioning from Azure Active Directory** 

The attributes `EmployeeHireDate`  but also `EmployeeLeaveDate`  are synchronized (via Cloud HR solutions) or has been manual updated on the pre-created work account in Azure AD. The attribute can be listed and [modified via Microsoft Graph API](https://learn.microsoft.com/en-us/graph/api/user-update?view=graph-rest-1.0&tabs=http):

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow3.png)

*Update of Employee Hire and Leave Date can be managed via Microsoft Graph API requests. Date must be in the format “YYYY-MM-DDThh:mm:ssZ”*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow4.png)

*Azure AD user blade shows `EmployeeHireDate`  (“Employee hire date”) in the properties. In addition, attributes such as `jobTitle` , `department` and `employeeId`  are maintained which will be used later in the provisioning process. At this time, the work account is disabled.*

**Synchronized work accounts: Provisioning from Active Directory via Azure AD Connect:**

The source system is able to synchronize `EmployeeHireDate` and `EmployeeLeaveDate`  of the user object (work account) to a custom attribute in Active Directory. Both attributes don’t exist in the Active Directory Schema, therefore you need to choose an attribute (e.g. `ExtensionAttribute1`). Azure AD Connect-Server and -Cloud Sync are configured to [synchronize the attributes to Azure AD](https://learn.microsoft.com/en-us/azure/active-directory/governance/how-to-lifecycle-workflow-sync-attributes#how-to-create-a-custom-synch-rule-in-azure-ad-connect-for-employeehiredate).

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow5.png)

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow6.png)

*The chosen “source attributes” in this sample are rather unsuitable. I was enforced to use them in this case because of limited availability of custom attributes in my environment (no Exchange custom attributes are available). In general, I would recommend using some custom / extension attributes.*

*Side Note: The msDS-cloudExtensionAttributes are not available in Cloud Sync. Currently, the [Microsoft Docs article about configuring entity mapping seems not accurate](https://github.com/MicrosoftDocs/azure-docs/issues/99588).*

### Workflow “Onboard pre-hire IT employee”

**Creating workflow from template**

The first workflow is designed to be executed for onboarding a pre-hire IT employee (seven days before `EmployeeHireDate`). I’ve used the built-in template “Onboard pre-hire employee” as basis to customize the workflow for IT employees.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow7.png)

**Execution Conditions**

The conditions for executing the workflow have been modified to the scope of users with the value “IT” in the property `department` :

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow8.png)

**Defined tasks in workflow**

The first two tasks are part of the workflow template and only the task name (yellow marked) has been modified. Those tasks are only related to the work account which has been already pre-created. I’ve added a “Custom task extension” on task order number 3 which will create a disabled privileged account for the pre-hire employee:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow9.png)

The Process to create a (disabled) privileged account is included, alongside of the standard tasks for the work account (e.g. “Generate TAP…” or “Send Welcome Mail”).

#### Custom task extension “Create-AADPrivilegedAccount”

Logic Apps can be triggered based on [custom task extension](https://learn.microsoft.com/en-us/azure/active-directory/governance/trigger-custom-task) and will be created right from the Lifecycle workflow blade. I’ve assigned the application permissions “User.Read.Write.All” to the managed identity of the Logic App.
*Side Note: Currently, there’s no option to assign application permissions or custom roles to limit permissions on create user accounts (User.Create). I hope there will be a support for AU-scoped creation of users ([similar to create a group in AUs](https://learn.microsoft.com/en-us/azure/active-directory/roles/admin-units-members-add#create-a-new-group-in-an-administrative-unit-1)) in the near future.* 

The logic app includes the following task steps:

**Phase 1: Get user details from work account**

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow10.png)

**Phase 2: Classify type of privileged account based on attribute**

I’ve chosen the design approach to implement a classification (already at the provisioning phase) to identify if the privileged account will be later used for managing Control- or Management Plane. You need to have an attribute which can be used to classify the intended use. In this case, I’m using the attribute `jobTitle`  which can be an indicator for the scope of privileged access (e.g. IAM Administrator often require Control Plane access).

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow11.png)

**Phase 3: Provisioning of Privileged Accounts**

The next action will create the privileged account with all required attributes including the `EmployeeHireDate`  as condition for the next Lifecycle workflow. `EmployeeId` will be used to generate the privileged account name (based on the naming convention). Furthermore, [“plus addressing” in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/plus-addressing-in-exchange-online) will be set to ensure the mail redirection to the work account (for PIM notification). This was a great tip on [Twitter by Merill Fernando](https://twitter.com/merill/status/1567337464788058113).

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow12.png)

But one important attribute for creating the account is still missing.
We need to generate a password which is required for our Graph API call.

There’s no built-in action to generate a complex password but a couple of ideas (from the community) to use expressions:

- [Generating pseudo-random passwords in Microsoft Flow | by Mark Stokes | Medium](https://medium.com/@mark_stokes/generating-pseudo-random-passwords-in-microsoft-flow-d7a9c232abe5)
- [Random Password Generator - Power Platform Community (microsoft.com)](https://powerusers.microsoft.com/t5/Power-Automate-Cookbook/Random-Password-Generator/td-p/1428897)

However, we need to take care on the value of the expression which needs to be protected and shouldn’t be visible in the workflow history or logs. Therefore, I’m using the expression within the HTTP action and enable “Secure Inputs and Outputs” in the settings:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow13.png)

This protects also the output value from the API response.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow14.png)

More details on [secure your logic apps](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-securing-a-logic-app?tabs=azure-portal#secure-data-in-run-history-by-using-obfuscation) are available from Microsoft Docs.

*Side Note: This sample shows the exclusion from the “Password expiration policy” and “no password change enforcement at next logon”. Microsoft recommends [ensuring that administrators has changed passwords](https://learn.microsoft.com/en-us/azure/active-directory/roles/security-planning#ensure-the-passwords-of-administrative-accounts-have-recently-changed) (at least 90 days). In my lab, I’m using Passwordless authentication methods and TAP only which is also enforced by Authentication Strength policies. Therefore, I’ve described and set no further scope on the user password management. But I would strongly recommend taking care on the initial password and further rotation and password protection.*

**Phase 4: Assignment of Custom Security Attributes**

After the account has been created successfully, the classification of the privileged account but also the relation to the work account will be stored in a custom security attribute (`associatedWorkAccount` and  `associatedPrivilegedAccount`) 

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow15.png)

The information about the relation between the user accounts will be stored as Object GUID reference (to the associated account) on both user objects:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow16.png)

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow17.png)

**Side Note: In this example I’ve used a multi-value field on the custom security attribute of the work account. This allows me to build relations to multiple privileged accounts (e.g. if account isolation to other environments or separated for Tier levels are required).**

It’s needed to assign the Azure AD admin role as “[Attribute Assignment Administrator](https://learn.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#attribute-assignment-administrator)” (58a13ea3-c632-46ae-9ee0-9c0d43cd7f3d) on the scope of the attribute set to the managed identity. This allows to assign and update the attributes of  `privilegedUsers` . The attribute set will be used for classification and account association only (no relation to Attribute-based Access Control use cases). 

```jsx
POST https://graph.microsoft.com/beta/roleManagement/directory/roleAssignments
Content-type: application/json

{
    "@odata.type": "#microsoft.graph.unifiedRoleAssignment",
    "roleDefinitionId": "58a13ea3-c632-46ae-9ee0-9c0d43cd7f3d",
    "principalId": "ObjectIdOfTheManagedIdentity",
    "directoryScopeId": "/attributeSets/privilegedUser"
}
```

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow18.png)

Managed Identity of the Logic App has a scoped “Attribute assignment administrator” role to the attribute set (based on `directoryScopeId` ).

### Workflow “Onboard new hire IT employee”

The workflow for enabling the pre-created accounts will be triggered on the defined `EmployeeHireDate` . I’ve spitted the workflows for onboarding the work and privileged account which gives me flexible options for different conditions and scopes.

**Creating workflow from template**

The process for enabling the work account has been created from the template “Onboard new hire employee” and will be also the basis for the process to enable the privileged account.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow19.png)

**Execution Conditions**

Conditions to trigger the workflows are set on `EmployeeHireDate`  and based on the same scope as the previous workflows (`department` equal “IT”)

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow20.png)

### Workflow: Onboard privileged account for new hire employee

**Creating workflow from template**

As already named, I’ve chosen the previous chosen template as basis, even the tasks are different from the previous workflow.

**Execution Conditions**

This workflow will be triggered on the `EmployeeHireDate`  which has been added during the process of creating the (disabled) privileged account.

**Defined tasks in workflow:**

There are only two tasks for onboard the privileged account for the new employee.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow21.png)

Generating the Temporary Access Pass (TAP) will be part of a custom extensions to send it to the account owner. There’s already a built-in task to generate a TAP and send the pass via email to user’s manager:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow22.png)

For the following scenario, I’ve preferred to choose a custom extension (Logic App) which gives you more flexible about the recipient and way how the TAP will be delivered. [Jan Bakker](https://twitter.com/janbakker_?ref_src=twsrc%5Egoogle%7Ctwcamp%5Eserp%7Ctwgr%5Eauthor) has written an [excellent blog post](https://janbakker.tech/automate-issuing-temporary-access-pass-for-joiners-with-lifecycle-workflows/) about this scenario and gives a detail description on an advanced scenario (using SMS and mail). I’m strongly recommend to read the article to learn more about his great solution! This allows the user to onboard their FIDO2 security key or Windows Hello for Business for password-less authentication.

Finally, a built-in task will enable the account in this workflow 

#### Custom task extension “Generate-AADPrivilegedAccountTAP”

The following Logic App has granted permissions to the Azure AD admin role “[Authentication Administrator](https://learn.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#authentication-administrator)” to create the TAP. I’ve created an Administrative Units (AU) which assigned the pre-created accounts based on a dynamic filter. This helps me to reduce the scope of this sensitive role to disabled (pre-created) privileged accounts only:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow23.png)

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow24.png)

*The pre-created privileged account for IT employees is assigned to the Administrative Unit “Tier1-ManagementPlane.OnOffBoarded”. All disabled privileged accounts (based on accountEnable attribute and naming convention) on the specific Enterprise Access (Tier) Level (shared attribute filter with classification from lifecycle workflow) are assigned to the AU.*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow25.png)

*Managed Identity of the Logic App has granted permissions on Administrative Unit-Scope only.
Privileged Accounts will be out of scope of this AU after they have been enabled and the dynamic filter has updated the user assignments.*

**Step 1: Get user details from pre-created account**

In the beginning of the workflow a variable need to be initialized for storing the mail address of the shared mailbox (for sending the TAP). Afterwards user details of the pre-created user account will be collected.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow26.png)

**Step 2: Create Temporary Access Pass for Initial Onboarding**

Microsoft Graph API can be used to create the TAP with specific parameters (such as lifetime or one time use). The action is also configured to use “[secure in- and output](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-securing-a-logic-app?tabs=azure-portal#secure-data-in-run-history-by-using-obfuscation)” to protect the secret of the TAP.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow27.png)

**Step 3: Send TAP from shared mailbox to the account owner**

In next step, delivery of the TAP (to the associated work account) will be defined. In this sample, I’m using a shared mailbox to send the pass via mail. Alternatively, you can also use other communication channel (Microsoft Teams, ticket system, password manager) and use another recipient.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow28.png)

#### Built-in task: Enable account

At this time, the account has no assignments to privileged roles or permission (such as Azure AD admin roles, role assignable or privileged access groups). This allows to use the “Authenticator Administrator” role and the default security context of “Identity Lifecycle Workflows” built-in tasks to modify the account. In the last step, the account will be enabled to complete the provisioning process.

### Workflow “Offboard an IT employee”

As already described, Lifecycle Workflows can be used to trigger the offboarding of accounts as part of the employee termination process. This can be done by updating the `EmployeeLeaveDate`  property via [Microsoft Graph API](https://learn.microsoft.com/en-us/graph/tutorial-lifecycle-workflows-set-employeeleavedatetime?tabs=http) or synchronizing the attribute from Azure AD Connect.

In this sample, I’m using a combined workflow to start the actions based on the `EmployeeLeaveDate`  attribute of the work account. This allows me to use a single trigger for offboarding of privileged account(s).

**Creating workflow from template**

There are two templates which can be used for (scheduled/planned) employee termination:

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow29.png)

Choose the right template which fits better to your scenario and to the right time when the privileged account should be offboarded. In addition, there’s also an “on-demand” workflow template if you like to trigger the employee termination manually or “in real-time” (short-term dismissal). 

**Execution Conditions**

In my example, I’m using the day of the `EmployeeLeaveDate` to trigger this workflow in scope of every employee which has the `department`  attribute set to “IT”.

**Defined tasks in workflow:**

Built-in tasks will be used in the offboarding workflow to disable the user account but also remove group memberships and access of the work account.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow30.png)

A custom extension with the name “Disable-AADPrivilegedAccount” takes care, that all associated privileged accounts will be disabled.

*Side Note: I would recommend the removal of privileged access by removing Access Package assignments instead of delete security group membership. In this sample, removing privileged access is not part of the workflow. I will try to include this topic in one of the next blog posts to show automation and implementation of access packages for lifecycle management of privileged access.*

#### Custom task extension “Disable-AADPrivilegedAccount”

**Step 1: Initialize variables for error handling and notification**

The first tasks will be used to initialize variables. Later on, they will be used if disabling the privileged account has been failed.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow31.png)

**Step 2: Using custom security attributes to get all associated privileged accounts**

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow32.png)

In the next steps, the custom security attribute of `associatedPrivilegedAccount`  needs to be collected for building a correlation to the other privileged account(s) which should be also terminated. The variable “privilegedAccount” will be initialized as “Array” to support multi-values from the custom security attribute.

**Step 3: Revoke sign-in sessions and disable account, send notification if disabling fails**

The foreach loop iterates over the list of associated privileged accounts.
Revocation of sign-in sessions and set `AccountEnabled` to “false” (Disable account) are the following actions. A notification to the previous named recipient (e.g. “Identity Operations Team”) will be sent in case the request to disable the account from Microsoft Graph has been failed

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-11-14-manage-privileged-identities-with-azuread-identity-governance/PrivUserLifeycleWorkflow33.png){:width="300px"}

Activities to disable and revoke sessions of privileged accounts needs extensive permissions.
User accounts with membership to privileged access/role-assignable groups or directory roles are particularly protected by Azure AD. More details are available from Microsoft Docs: [Azure AD built-in roles - Who can perform sensitive actions - Microsoft Entra](https://learn.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#who-can-perform-sensitive-actions)

Unfortunately, it’s needed to assign sensitive permissions to the Logic App if you want to automate this process. Assignments of the “Privileged Authenticator Admin” directory role to the Managed Identity are needed to disable those accounts. Alternatively, you need to remove all privileged group or access package assignments of the account before this workflow will be executed.
Moving to a half-automated process for disabling high-privileged accounts (create ticket to disable account) can be another option. Currently, there’s no way to build an Administrative Unit to delegate the “Privileged Authenticator Admin” role on a limited scope.

## Security considerations to avoid escalation paths

As already mentioned, the used Custom Extensions (Logic Apps) are using high-privileged permissions. In addition, the Azure AD admin role “Lifecycle Workflows Administrator” delegates permission to modify the workflows. Therefore, I can only strongly recommend to consider them as high-sensitive assets. They should be part of your security concept to protect Control plane (Tier0) management assets. Furthermore, make sure you protect the Logic App workflows incl. HTTP trigger and code. Christropher Brumm has written an excellent blog post about [securing Logic Apps](https://chris-brumm.medium.com/logic-apps-and-azure-active-directory-a3857c9e3951). 

## Post-onboarding tasks and processes

### Optional: Verified ID for Employees with Privileged Accounts

Issuing the Verified ID should be separated from the previous named “Identity Lifecycle” process.

### Request and assignment of privileged access

Azure Portal and other privileged interfaces or intermediaries will not be accessible this stage. Because the user is part of Conditional Access policies which blocks access from non-privileged devices and any user access without assignments to administrative groups. Only access to request access packages (from ”My Access portal”) will be allowed. The access package assignments are also be used to add the privileged account in the user assignment of Conditional Access for Privileged Accounts. This will be the topic for the next part of the blog series about using “Identity Governance” as Entitlement Management for Privileged Access.