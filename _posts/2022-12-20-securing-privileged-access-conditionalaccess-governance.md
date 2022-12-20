---
title: "Securing privileged user access with Azure AD Conditional Access and Identity Governance"
excerpt: "Conditional Access and Entitlement Management plays an essential role to apply Zero Trust principles of “Verify explicitly“ and “Use least-privilege access“ to Privileged Identity and Access. In this article, I like to describe, how this features can be use to secure access to privileged interfaces and how to assign privileged access by considering Identity Governance policies."
header:
  overlay_image: /assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/IdentityGovCaPoliciesTeaser.png
  overlay_filter: rgba(102, 102, 153, 0.85)
  teaser: /assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/IdentityGovCaPoliciesTeaser.png
search: true
toc: true
toc_sticky: true
categories:
  - Azure AD
tags:
  - AzureAD
  - PrivilegedIAM
  - IdentityGovernance 
last_modified_at: 2022-12-20
---

# Securing privileged user access with Azure AD Conditional Access and Identity Governance

*Conditional Access and Entitlement Management plays an essential role to apply Zero Trust principles of "Verify explicitly" and "Use least-privilege access" to Privileged Identity and Access. In this article, I would like to describe how this features can be used to secure access to privileged interfaces and how to assign privileged access by considering Identity Governance policies.*

## Provisioning of Privileged Identity

In the [previous blog post](https://www.cloud-architekt.net/manage-privileged-identities-with-azuread-identity-governance/), I’ve described how to onboard a new privileged user account by using  "Lifecycle Workflows" in Azure Active Directory. As a result of the provisioning process, a dedicated privileged account has been created which the following configurations and properties:

- Cloud-only account without any privileges by default
    - Relation between work/privileged account and scope of access level (Control-/Management Plane) are stored in custom security attributes
    - Privileged user account has no assigned RBAC roles in Azure AD, Azure or any other platform per default (at this time, only unprivileged and [default member user permissions](https://learn.microsoft.com/en-us/azure/active-directory/fundamentals/users-default-permissions?WT.mc_id=AZ-MVP-5003945#compare-member-and-guest-default-permissions) are available for this dedicated account)
- Assignment to Administrative Unit (AU) for Privileged Accounts (by using dynamic membership filter)
- Assignment to Group which contains all Privileged Accounts that has been provisioned for the targeted Enterprise Access/Tier level (in this case: dug_AAD.Tier1.PrivilegedAccounts)
- Optional: Employee owns Verified Credential as employee of company’s IT department
- Employee has received Temporary Access Pass for passwordless onboarding

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm.png)

_Overview of work and privileged account provisioning. In this article, I will describe the implementation that governs the process for privileged access assignments and enforcing access controls in context of privileged access with Conditional Access Policies._

## Principles of Tiered Administration Model
The goal is to provide an approach which supports the principles of tiering model and avoid unauthorized access paths by establishing security boundaries. Furthermore, the implementation of the following role-based access and persona model should allows us to identify, monitor, and govern sensitive privileged accounts.

In summary, the following objectives will be try to be addressed:
- Isolation between Privileged and User Access, Work accounts ("User Access Plane") have no option to request privilege access and interfaces
- Tiered administration level of three different personas:
  - "Control Plane": Dedicated accounts for Identity Administrators with access to critical assets in Azure AD (such as Conditional Access, PIM) and Microsoft Azure (Authorization, Identity SOAR, Domain Controllers) which has direct or indirect impact of the (identity-driven) perimeter. This includes also the definition and management of Identity Governance, Privileged Identity and Access.
  - "Management Plane": Horizontal access to manage resources and Microsoft Azure as cloud platform. No access to higher tier. Approves access requests to lower tier on pre-defined policies and permission scope.
  - "Workload Plane": DevOps accounts has limited access on resource-level to manage deployed services within defined guard rails (such as Azure Policies, compliance and security checks in pipelines). Access to privileged interfaces requires a secured corporate device and will be particularly monitored.
- Central management of assignments to security, access and governance policies (by using role groups)
- Preventive measures to avoid assignment of privileged roles to different access levels (breaching tiering access model)
- Definition of governance process as policy to gain access
- Zero standing access and no assigned default permissions (after account provisioning)
- Continuous review and renewal of assigned (eligible) roles
- Comprehensive audit coverage to track changes of privileged access
- Option to check "separation of duties" and self-service for request/renewal of access
- Automated response to identity and device threats in Zero Trust policy engine (Conditional Access)
- Access outside of pre-defined role groups will be blocked by access policies
- Enforce using [device profiles](https://learn.microsoft.com/en-us/security/compass/privileged-access-devices#device-roles-and-profiles?WT.mc_id=AZ-MVP-5003945) and authentication methods with high security standards

<a href="{{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/IdentityGovCaPoliciesOverview2.png"><img src="{{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/IdentityGovCaPoliciesOverview2.png" alt="Overview" /></a>
_Overview of personas in a "Tiered Administration Model" and their requests to gain privileged access with applied policies._


## Privileged user journey after onboarding

### First sign-in of privileged user on PAW device

At this time, the privileged user is able to use the (one-time) Temporary Access Pass (TAP) to configure the Privileged Access Workstation (PAW) and register a FIDO2 security key or Windows Hello for Business (WHfB) during the device setup.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm1.png){:width="60%"}

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm2.png){:width="60%"}

_TAP will be used during the PAW self-provisioning process to set up passwordless and phishing-resistant authentication method (in this case, Windows Hello for Business)._

Conditional Access for Privileged Users are assigned to members of privileged role groups in the following scenarios. Enforcing phishing-resistant authentication methods or filtering PAW device cannot be enforced before device setup has been completed.

Therefore, TAP must be allowed as authentication strength for privileged users for the onboarding process. Currently it’s not possible to exclude "My sign-ins" portal as "Cloud App" from Conditional Access to allow using TAP for registration of phishing-resistant authentication methods only. You need to include TAP to your authentication strength during the time of onboarding.

*Consider to [enable Web sign-in in Policy CSP (Authentication)](https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-authentication?WT.mc_id=AZ-MVP-5003945#authentication-enablewebsignin) if you want to use TAP from the Windows logon on Azure AD-joined devices.*

Intune Device Configuration and Policies for PAW devices will be assigned to all privileged users. This allows policies to apply right after device provisioning (including highly restricted web and application access).

### Using MyAccess portal to request privileged roles

As already mentioned, the privileged user has no assignment to privileged roles yet. Therefore, access to privileged interfaces (such as Azure Portal) will be also blocked for this account (in Conditional Access) as long no membership to a privileged role group or particular exclusion exists. Only access to whitelisted URLs is granted (by using "[local URL lock proxy](https://learn.microsoft.com/en-us/security/compass/privileged-access-deployment?WT.mc_id=AZ-MVP-5003945#url-lock-proxy)") which blocks public internet or other cloud app access.

At next, the new privileged user has the option to request an access package to gain membership to specific privileged roles. Only access packages with privileged roles in relation to the classification of the privileged account are available. In this case, the privilege account has been created for a team member of the "Azure Infrastructure Team" and classified as "Management Plane" (Tier1) administrator during provisioning process. Therefore, only access packages for those level of privileges will be shown (e.g., PlatformOps or NetOps groups to gain RBAC assignment in Microsoft Azure).

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm3.png)

In near future, it would be possible to force the requester to verify the identity with a verifiable credential (as employee of the company IT). This allows explicitly verifying the user identity by using this separate verification method. Risks on "account takeover" during the onboarding or provisioning process ("dormant account") can be reduced with requesting an identity which is owned by the employee ("account owner").

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm4.png){:width="30%"}

Enforcing user verification (with Microsoft Entra Verified ID) when requesting access packages will be available soon (currently in private preview). Source: [Twitter (@MrDebChoudhury)](https://twitter.com/MrDebChoudhury/status/1558174911520157697)

*Side Note: Additional advanced scenarios can be achieved by [trigger a custom logic app](https://learn.microsoft.com/en-us/azure/active-directory/governance/entitlement-management-logic-apps-integration?WT.mc_id=AZ-MVP-5003945) after an assignment request or approval has been made. This could include requests to check detailed employee information (user details matches to privilege role profile) or requesting identity security status.*

As defined in the access package policy, the request needs to be approved by another privileged user according to the requirement of the certain privileged access level.

After approval and assignment of the access package, the privileged user has granted eligible assignments to the RBAC role (in Azure PIM) and will be covered by particular Conditional Access policies.

Let’s have a closer look at the configuration in Identity Governance and Conditional Access to govern and secure the privileged access…

## Identity Governance Access Packages for Privileged Roles

In the following samples, high-privileged permissions in Azure, Azure AD but also Azure DevOps will be assigned on membership in "[Role-assignable groups](https://learn.microsoft.com/en-us/azure/active-directory/roles/groups-concept?WT.mc_id=AZ-MVP-5003945)" only. Assignment as member can be requested as access packages by privileged users which allows to govern a lifecycle process for privileged access. Access catalogs exists for the different levels of the [Enterprise Access Model](https://aka.ms/SPA). 

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm5.png)

*Overview of catalogs with different role-assignable groups for privileged access in Azure AD, Azure, Azure DevOps and Microsoft 365 (spitted between Control-, Management- and Workload Plane).*

### Privileged Access Packages and Request Approval

In the previous example, the privileged user has requested an access package to become role permissions as "PlatformOps". The policy defines the following requirements to get an assignment to this access package.

- Only privileged users on "Management Plane" (Tier1) are able to request assignment (no upper/lower access from work account or "Control Plane" (Tier0) administrators.
- Access package policy needs to be approved by same or higher privileged user (in this case, Identity Operations)
- Approver needs to enter a justification to document validation and authorization checks
- Optional: Using Verified ID to explicitly verify the identity of the requester after onboarding

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm6.png)

*Approval can be configured in the policy of the access package to define the approver and other parameter of this process step. This includes also the advanced configuration for [multi-stage approvals](https://learn.microsoft.com/en-us/azure/active-directory/governance/entitlement-management-access-package-approval-policy#multi-stage-approval) or [alternate approvers](https://learn.microsoft.com/en-us/azure/active-directory/governance/entitlement-management-access-package-approval-policy#alternate-approvers).*

In addition, the policy also defines the lifecycle of the assigned access package. In this scenario, assignment needs to be renewed within one year and a quarterly review is needed by members of the Identity Operations Team (also approver of the access package).

*Side Note: Delegation of the access review could be also delegated to a specific group of member(s) within the department or business of the assigned privileged user.*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm7.png)

*Access Review can be configured as part of the policy of the access package.*

### Using Role-assignable groups (PRG) for role-based access

Microsoft has introduced "role-assignable groups" to assign Azure AD directory roles to security groups. Those group type are [particular protected in Azure AD](https://learn.microsoft.com/en-us/azure/active-directory/roles/groups-concept#how-are-role-assignable-groups-protected) and also a prerequisite to use the capabilities of [Privileged Access Group](https://learn.microsoft.com/en-us/azure/active-directory/privileged-identity-management/groups-features) (PAG) in Azure PIM.

PAGs allow to assign eligible membership to security groups which can be used to provide just-in-time (JIT) access for group-based RBAC assignments outside of the supported PIM roles (in Azure and Azure AD). In general, this works for any application whose authorization can be assigned to a security group in Azure AD.

In a nutshell, role-assignable groups can be used to assign members for Azure AD directory role but it allows also to assign them as eligible group member/owner of this group type when Privileged Access Group is enabled. More details about the [differences between Role-assignable and Privileged Access groups](https://learn.microsoft.com/en-us/azure/active-directory/privileged-identity-management/concept-privileged-access-versus-role-assignable?WT.mc_id=AZ-MVP-5003945) are very well described by Microsoft.

*Side Note: Consider [unsupported scenarios and known issues](https://learn.microsoft.com/en-us/azure/active-directory/roles/groups-concept) but also [limitations of maximal numbers of role-assignable groups](https://learn.microsoft.com/en-us/azure/active-directory/enterprise-users/directory-service-limits-restrictions?WT.mc_id=AZ-MVP-5003945).*

I’m using role-assignable groups to create role groups (prefix "PRG") for Azure AD, Azure and Azure DevOps (in this example, named as "prg_Lab-Tier1.Azure.1.PlatformOps"). This allows to avoid modification from "Group Admins" (incl. service-specific roles, e.g., Intune Admin) and provides [restricted user management for group members to specific privileged roles only](https://learn.microsoft.com/en-us/azure/active-directory/roles/permissions-reference?WT.mc_id=AZ-MVP-5003945#who-can-perform-sensitive-actions). 

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm8.png)

*Access Packages assigns requester to a role-assignable group after the approval and requirements of the policy have been satisfied. This group will be used to assign roles in Azure AD- and Azure-RBAC but also membership to security groups in Azure DevOps or any other group-based authorization.*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm9.png)

*Privileged users are assigned as permanent members to a role-assignable group which will be used for (eligible) assignment to the privileged roles in the target RBAC system. Even if the privileged user is permanent member of this group, no standing privileged access will be assigned based on this membership. As group member, the user is able to request eligible roles in Azure and Azure AD or group membership to a PAG via PIM. For example, admMOB1 is permanent member of the shown role group but has to request the eligible assignment to the Azure RBAC role via Azure PIM.*

### Eligible Assignment in Privileged Access Group

Unfortunately, access packages cannot be used to assign eligible membership to Privileged Access Groups (PAG) yet. This is also a reason why I’m using a role group (PRG) which will be assigned to the PAG as member. This nested group membership allows me to take benefit from using access packages in combination with PAGs.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm10.png)

*Role group will be assigned as eligible member of a PAG. Nesting of this role-assignable groups is necessary because of missing support to assign eligible membership to a PAG in access packages.*

### Access Review of Assignments to Groups or Access Packages

As already described, the initial policy can be used to configure an access review for assignments on access packages. It’s also possible to create an access review for the role group only. Reviewer will use the "My Access" portal to review the assignments including recommendations and useful remarks regarding assigned users.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm11.png)

*Access Review shows recommendation to remove assignment (deny continued access) based on the activity of the assigned user (last sign-in).*

### Authorized elevation path for DevOps accounts

It’s needed to provide DevOps engineers also limited access to Azure resources and Azure DevOps. Access should be very restricted on workload/data plane (reader permission or data action of individual/scoped resources) or Dev/Test lab environments. In addition, a managed and secured CI/CD pipeline should be used to deploy resources (as infrastructure-as-code) and isolated from contributor permissions in Azure DevOps. Nevertheless, DevOps employees are mostly using no Privileged Admin Workstation (PAW) or a dedicated privileged user account. 

Therefore, you should particularly consider those user account and access requests with Conditional Access and Entitlement Management as well. Avoid options to gain access of overprivileged roles or in scope of "Management Plane". All DevOps users with approved access requests on restricted "Workload Plane" will be assigned to a security group which can be used to enforce Conditional Access Policies to verify additional conditions and controls on user account and device.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm12.png)

*DevOps accounts are able to request restricted access on "Data/Workload plane" on scoped resources in Azure and limited Azure DevOps Contributor role (without permissions to modify governance or protection policies in repo and pipelines) of a project.*

## Designing Conditional Access for Privileged Access

The previously described role groups will be used to scope the Conditional Access policies for securing privilege user access. I can only strongly recommend to consider the privileged accounts as personas in your policy design to enforce higher requirements for grant (privileged) access. Microsoft Docs shows an excellent example how you can [structure your policies based on personas](https://learn.microsoft.com/en-us/azure/architecture/guide/security/conditional-access-architecture?WT.mc_id=AZ-MVP-5003945#conditional-access-personas) and the resources which being accessed. An [access template card](https://learn.microsoft.com/en-us/azure/architecture/guide/security/conditional-access-architecture?WT.mc_id=AZ-MVP-5003945#access-template-cards) could help you to define the characteristics of each persona.

In my scenario, I’ve used the following approach to visualize the requirements for a target policy set. This includes all conditions and signals but also controls which are available and should be applied for every access to a privileged intermediary, interface or resource. 

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm13.png)

*Overview of authorized access paths from Privileged and DevOps Accounts including their conditions and control options for Conditional Access.*

### Assignment to privileged account vs. privileged role group

Some of the policies cannot be enforced directly after the provisioning of a privileged user because of some technical dependencies (e.g., enforce WHfB/FIDO2 before onboarding).
Therefore, you should consider to apply these policies when the privileged user has assigned privileged roles (after provisioning). In this sample, I’ve configured the user assignment of the policy to the privileged role group instead of applying them to all privileged accounts (via dynamic group which contains all accounts). Nevertheless, policies to enforce MFA and compliant devices will be applied at all times.

## Conditional Access Policies for Privileged Users with (Eligible) Roles

The following policy design has been defined to protect every privileged user account with an assignment to a role group. This is an example and should defined your own policy set based on your security or governance requirements. The policies will be applied when accessing any Azure AD-integrated resource (target assignment to "all cloud apps").

[Alex Filipin](https://twitter.com/alexfilipin) has published very good [policy templates for admin protection](https://github.com/AlexFilipin/ConditionalAccess/tree/master/PolicySets/Category%20structure%20for%20AADP2) last year. Based on his work, I’ve developed this policies further and adjusted to my scenario.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm14.png)

*Overview of Conditional Access Policies for Privileged user accounts.*

### Enforce Device Security and PAW

- Policy 120: Access of Privileged accounts is restricted to Privileged Admin Workstation (PAW) only.
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm15.png)
    
    *Device Filter will be used to restrict access for PAW only. Keep in mind, some Azure AD directory roles are to modify ExtensionAttributes of Device-object (including Intune Admin).*
    
- Policy 123: Only Windows Devices with compliance status and low machine risks in Intune are allowed to use for privileged access.
*Example, PAW [Device Compliance of Privileged Access deployment](https://learn.microsoft.com/en-us/security/compass/privileged-access-deployment?WT.mc_id=AZ-MVP-5003945#finish-workstation-profile-hardening) from Microsoft Docs.*

### Enforce always strong authentication and risk evaluation

- Policy 121: Access from privileged accounts must be blocked when user risk is high
- Policy 122: Phishing-resistant authentication methods (such as FIDO2, WHfB) can only be used
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm16.png)
    
    *Authentication Strengths must be used to enforce FIDO2 Security Key or "Windows Hello for Business" as Phishing-Resistant MFA option for all cloud apps.*
    
- Policy 124: Access to non-privileged interfaces (productivity apps such as Office 365) should be monitored and restricted by using "Conditional Access App Control" in Microsoft Defender for Cloud Apps (MDA). And even "[local URL lock proxy](https://learn.microsoft.com/en-us/security/compass/privileged-access-deployment?WT.mc_id=AZ-MVP-5003945#url-lock-proxy)") will block unauthorized cloud apps access, it's an additional layer for managing partly sanctioned apps.
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm17.png)
    
    *Cloud Apps outside of Privileged Intermediaries or Interfaces should be monitored to detect access to productivity workloads (breach of tiering level to "User Access" and separation to "Work account").*
    

The following policy is already configured to apply for all user accounts in the Azure AD tenant:

- [Re-authentication](https://learn.microsoft.com/en-us/azure/active-directory/conditional-access/howto-conditional-access-session-lifetime#require-reauthentication-every-time) will be enforced when sign-in risk is medium or higher
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm18.png)
    
    *Remediation of sign-in risk with MFA requirement, satisfied by claim in the token, isn’t enough in my opinion. This allows attackers to use MFA claim in token replay scenarios. Therefore, I’m enforcing re-authentication in case of a sign-in risk.*
    

In addition, you can also configure the following policy to manage SSO behavior and sign-in frequency setting of the privileged account:

- Never persistent (SSO) browser session with re-authentication after 10 hours (work day).
    
    ![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm19.png)
    
    *Restrict authentication session for privileged account limits the duration of token and limit time window for abusing stolen token (from replay attacks).*
    

### Secure registration of MFA and Azure AD devices

Registration of security information and register Azure AD devices should be particularly protected. Temporary Access Pass (TAP) satisfies requirements for MFA. Therefore, both user actions should be configured to require MFA which enforces the use of TAP during the onboarding process. This is one of the policy which also applies to all different types of users (including work accounts). More details can be found on [Microsoft Docs](https://learn.microsoft.com/en-us/azure/active-directory/conditional-access/howto-conditional-access-policy-registration?WT.mc_id=AZ-MVP-5003945).

## Verify access explicitly to Privileged Interfaces and Intermediaries with Conditional Access

In addition to protect privileged accounts, I’ve configured also a policy set to protect the privileged intermediaries and interfaces explicitly. This allows also to enforce separation of user and privileged access paths to avoid unintended and granted access from work accounts (on "User Access" plane) to "Control- or Management Plane". This includes also to consider the DevOps Accounts for accessing Azure Portal and Azure DevOps for restricted access.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm20.png)

"*Securing Privilege Access" documentation from Microsoft describes isolated virtual zones which should be considered in the Conditional Access implementation. Image source: "[Securing privileged access](https://learn.microsoft.com/en-us/security/compass/overview?WT.mc_id=AZ-MVP-5003945)" from Microsoft Docs.*

## CA Policy Assignment on accessing Privileged Intermediaries and Interfaces

A set of conditions and controls should apply to all privileged cloud apps, regardless who try to gain access. For example, Microsoft recommends [requiring MFA for any user accessing Azure Management](https://learn.microsoft.com/en-us/azure/active-directory/conditional-access/howto-conditional-access-policy-azure-management?WT.mc_id=AZ-MVP-5003945).

Those policies will also cover authorized access paths from DevOps accounts which have restricted access to Azure Management and Azure DevOps. Nevertheless, the DevOps users need to satisfy the conditions by using a specialized device with higher security standards than a default workstation.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm21.png)

*Overview of my configured Conditional Access Policies to protect every access to a cloud app which will be used for Control- or Management Plane.
Side Note: Keep in mind, exclude emergency access accounts ("Break-glass") and Azure AD synchronization accounts from this policies.*

The following policies will be enforced and needs to be satisfied for granting access:

- Policy 300: Access from every work account, privileged accounts without any role group membership (e.g., during onboarding process) and unauthorized DevOps accounts (without approved access package) will be blocked
- Policy 301/304: Only secured endpoints and authorized device profiles can be used for privileged sessions ([PAW and Specialized Devices](https://learn.microsoft.com/en-us/security/compass/privileged-access-devices?WT.mc_id=AZ-MVP-5003945#device-roles-and-profiles)) and requires a healthy compliance status in Microsoft Intune.
- Policy 302: Any user account with high user risk will be blocked from getting access
- Policy 303: Phishing-resistant authentication methods needs to be used by any type of user account
- Policy 305: Restricted access from DevOps users can be particularly monitored in Microsoft Defender for Cloud Apps by using session policies.

### Assignment to "Azure Management" in Cloud Apps

Azure Management portal and API services should be selected as "cloud app" for the policy to protect privileged interfaces. However, the policy will be enforced for a set of services which are bounded to the Azure portal and the API. Please consider the [list of known dependencies from Microsoft Docs](https://learn.microsoft.com/en-us/azure/active-directory/conditional-access/concept-conditional-access-cloud-apps?WT.mc_id=AZ-MVP-5003945#microsoft-azure-management) which has an indirect impact to such a policy assignment.

Azure DevOps is one of the related cloud apps. This is another reason why I included the DevOps accounts as part of the authorized access path.

### Using App Filtering for Cloud App Assignment

In addition to "Azure Management", other cloud apps need to be protected as privileged interface. This includes the following well-known Enterprise Apps for programmatically access with sensitive contested and delegated API permissions:

- Microsoft Graph Explorer
- Microsoft Graph PowerShell
- Microsoft Intune PowerShell

Custom Security attributes can be used to classify "Enterprise Apps" in relation to the "Tier Level" of the Enterprise Access Model.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm22.png)

*Attribute set of "privilegedInterface" is assigned to set classification of the "Microsoft Graph PowerShell" as sensitive application which should be protected as Control/Management Plane asset.*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm23.png)

*App Filters allow us to assign "Cloud Apps" which matches the attribute name and values from Custom security attributes. In this case, policy applies to all access to classified "Control and Management Plane" apps.*

### Restrict access for non-supported CA target apps

Some sensitive portals and apps cannot be selected in the CA cloud app assignment. As far as I know, this includes also the "Microsoft 365 Defender" portal.

The following M365 Defender (legacy) portals can be selected in a policy which allows to block access from non-privileged accounts:

- Microsoft Defender for Cloud Apps
(Cloud app name: Microsoft Cloud App Security)
https://TenantName.portal.cloudappsecurity.com
- Microsoft Defender for Endpoint
(Cloud app name: Windows Defender ATP’)
https://securitycenter.microsoft.com
- Microsoft Defender for Identity
(Cloud app name: Azure Advanced Threat Protection)
https://TenantName.atp.azure.com

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm24.png)

*After blocking access to the legacy portals, access to Microsoft 365 Defender portal (https://security.microsoft.com) seems still possible even an error message shown blocked access to back-end calls to other portal APIs.* 

Block access to such non-supported CA target apps could be implemented by using CA App Control and access policies in Microsoft Defender for Cloud Apps.

*Side Note: [thinformatics](https://blog.thinformatics.com/2021/08/how-to-restrict-access-to-microsoft-365-defender-apps/) has been published a blog post to restrict access to M365 Defender portals for members of a Privileged Access Group.*

## Security considerations to avoid escalation paths

All configuration items to protect assignment of privileged access and enforce Conditional Access policies are highly critical and require particular attention. The following delegated Azure AD roles allows to manage any aspects of the associated identity governance and security configuration:

- Conditional Access Administrator
- Global Administrator
- Identity Governance Administrator
- Privileged Role Administrator

Many other Azure AD role members will be able to exclude their accounts from Conditional Access policies when you are using security groups for managing policy exclusions (such as "Emergency Access" or "Synchronization Accounts").

Ownership can be assigned to "Role-assignable groups" which allows to change the sensitive groups outside of assigned Global Administrators and Privileged Role Administrators.

### Identity Governance Delegation and Role-Assignable Groups

Review and consider catalog ownership and access package delegations in Identity Governance.
For example, a delegated access package manager is able to create access package and assigns role-assignable membership to their account to become eligible for Azure AD or Azure RBAC roles.

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm25.png)

*Session Catalog includes role-assignable group with sensitive assignment to Azure AD admin role "Global Admin", delegation for managing access package is delegated to a work account (breach of tier level)*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm26.png)

*Access package manager creates new access package and assign the included catalog resource (membership to role-assignable group with Global Admin assignment) to their own account or any other user.*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm27.png)

*Assignment to the role group will be executed from "Azure AD Identity Governance" as actor which has full access to change membership on protected role-assignable groups (alongside of Global/Privileged Role Admin and Group Owner).*

![Untitled]({{ site.url }}{{ site.baseurl }}/assets/images/2022-12-20-securing-privileged-access-conditionalaccess-governance/PrivUserCaElm28.png)

*Adding role-assignable groups to session catalog is restricted to Global-/Privileged Role Admin and Owner of the group. But creation and assignment of access package with role-assignable group isn't restricted to this roles. Therefore, take care on delegated roles in Identity Governance and all created access packages from sensitive catalogs.*

Microsoft has given the following caution on the docs article about [managing access to resources in entitlement management](https://learn.microsoft.com/en-us/azure/active-directory/governance/entitlement-management-access-package-first?WT.mc_id=AZ-MVP-5003945#step-2-create-an-access-package):

> The *[role-assignable groups](https://learn.microsoft.com/en-us/azure/active-directory/roles/groups-concept?WT.mc_id=AZ-MVP-5003945)*  added to an access package will be indicated using the Sub Type *Assignable to roles*. For more information, check out the *[Create a role-assignable group](https://learn.microsoft.com/en-us/azure/active-directory/roles/groups-create-eligible?WT.mc_id=AZ-MVP-5003945)*
 article. Keep in mind that once a role-assignable group is present in an access package catalog, administrative users who are able to manage in entitlement management, including global administrators, user administrators and catalog owners of the catalog, will be able to control the access packages in the catalog, allowing them to choose who can be added to those groups. If you don't see a role-assignable group that you want to add or you are unable to add it, make sure you have the required Azure AD role and entitlement management role to perform this operation. You might need to ask someone with the required roles to add the resource to your catalog. For more information, see *[Required roles to add resources to a catalog](https://learn.microsoft.com/en-us/azure/active-directory/governance/entitlement-management-delegate#required-roles-to-add-resources-to-a-catalog)*