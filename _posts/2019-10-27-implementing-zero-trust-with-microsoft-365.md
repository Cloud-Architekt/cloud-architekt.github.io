---
layout: post
title:  "Implementing Zero Trust with Microsoft 365 (Study collection)"
author: thomas
categories: [ Azure, Security, AzureAD ]
tags: [azure, azuread, security]
image: https://www.cloud-architekt.net/assets/images/zero-trust-network.jpg
description: "Do you want to know how to implement a ZeroTrust security model with Microsoft 365? Take a look on my link collection to learn more about the modern approach to cybersecurity."
featured: true
hidden: false
---

Building and implementing “Zero Trust networks” (ZTN) is essential to archive a new cyber security model in a world of modern IT and cloud transformation. Previous (traditional) perimeter-based network security depend on the paradigm that data is secure as long as it is located on-premises and within local (corporate) networks. This doesn’t fit in today’s business environment of cross-business collaboration, usage of (personal) mobile devices, specific protection of data security and cloud apps. 

It’s a mind shift from a model of implicit trust to one of explicit verification.
In a brief summary this means: **“Always Verify. Never Trusts”**.

Microsoft 365 includes features and services to support this approach to validate every objective in your environment such as the following one:

* Users (Azure Active Directory)
* Applications (Microsoft Cloud App Security)
* Devices and Endpoints (Microsoft Intune and Microsoft Defender ATP)
* Data (Microsoft Information Protection)

Various Microsoft partner are also providing extended solutions that could be an essential part of your “Zero Trust” implementation (for example to replace network segmentation with app segmentation or securely publish legacy apps).

It is advisable, however, to take a closer look on (conceptual) approaches of other cloud provider (such as Google) to see differences or common position and standards (for your multi-cloud or exit strategies).

There are already many excellent study resources that I like to share with you.
In this blog post you’ll find few links to white papers, documentations and session recordings to learn more about “Zero Trust”.

# Overview of "Zero Trust" approach
[“A new era of security”](https://query.prod.cms.rt.microsoft.com/cms/api/am/binary/RE3YnRL)  
High-level overview of Microsoft's view on “Zero Trust” security model, dual perimeter strategy and the three pillars to achieve the goal: Signal, Decision and Enforcement.

[Traditional perimeter-based network defense is obsolete—transform to a Zero Trust model](https://www.microsoft.com/security/blog/2019/10/23/perimeter-based-network-defense-transform-zero-trust-model/)  
Comparison of the various stages of transformation from traditional to “optimal” (fully) implemented model.

[Zero Trust Maturity Model](https://aka.ms/Zero-Trust-Vision)  
Guiding principles and the maturity model are in focus of this white paper.

# Building “Zero Trust” with Microsoft 365
[Achieve Zero Trust with Azure AD conditional access](https://aka.ms/ZeroTrust)  
Microsoft’s resource page of all ZTN resources related to Azure AD and conditional access.

[Implementing a Zero Trust approach with Azure Active Directory](https://www.microsoftpartnercommunity.com/atvwr79957/attachments/atvwr79957/Australia_Security_Compliance_Practice/10/1/Implementing-a-Zero-Trust-approach-with-Azure-Active-Directory.pdf)  
This white paper is written to to discuss how to translate the principles of “Zero Trust” concretely with Azure AD and other Microsoft security products for your organization.

[“Zero Trust - Identity approach” by Brjann Brekkan](https://www.linkedin.com/posts/brjann-brekkan-0a24a72_identity-and-zero-trust-strategy-techdays-ugcPost-6593043152322605056-elDz)  
Slides from Brjann Brekaan’s session about the Identity approaches in “Zero Trust” scenarios at the TechDays Sweden 2019. Overview of mindset, methods and Microsoft’s related identity security solutions.

[Zero Trust part 1: Identity and access management](https://www.microsoft.com/security/blog/2018/12/17/zero-trust-part-1-identity-and-access-management/)  
Blog series about the implementation in Microsoft 365 solutions. This part of the series describes in general the approach of Zero Trust approach in Microsoft’s Azure AD.

[“Zer0ing Trust - Do Zero Trust Approaches Deliver Real Security” by David Weston](https://github.com/dwizzzle/Presentations/blob/master/David%20Weston%20-%20Zer0ing%20Trust%20-%20Do%20Zero%20Trust%20Approaches%20Deliver%20Real%20Security.pdf)  
David Weston placed this question in his talk at the Blackhat USA in 2018. Includes details of device security and “device attestation” on iOS and Android.

## Session Recordings
[No More Firewalls! How Zero-Trust Networks Are Reshaping Cybersecurity (RSA 2019)](https://www.youtube.com/watch?v=pyyd_OXHucI)  
Matt Soseman, Security Architect at Microsoft, gives an excellent impression of various features and real-word scenarios which will be covered in a full-implemented “Zero Trust” approach with Microsoft 365. This session makes clear how Microsoft likes to mesh the several products from Azure AD, MCAS up to Intune.

[Taking steps three two and one to zero-trust (Ignite 2018)](https://www.youtube.com/watch?v=LLfeuCK7fJ4)  
Alex Weinert shows in his Ignite session how you can start your transformation journey with protecting people, devices, data and workloads by Microsoft 365 security features.

[Azure AD Conditional Access and Enabling Zero Trust (Ignite 2018)](https://www.youtube.com/watch?v=XruceejcCKQ)  
Mechanics session from Ignite 2018 about the important role of “Conditional Access Policies” as part of your ZTN strategy.

[[Zero Trust Access and IAM by Michael Dubinsky (HIPC2019)](https://www.brighttalk.com/webcast/17685/362278/zero-trust-access-and-iam)  
Awesome talk by Michael Dubinsky with focus on the various “blocks” to establish a full covered Zero Trust implementation. Special consideration of  IAM as key role.

# Showcases and real-world scenarios of “Zero Trust”
[Implementing a Zero Trust security model at Microsoft](https://www.microsoft.com/en-us/itshowcase/implementing-a-zero-trust-security-model-at-microsoft)  
Description of Microsoft’s internal IT journey to implement a layered approach to secure corporate and customer data.

[Why banks are adopting a modern approach to cybersecurity—the Zero Trust model](https://www.microsoft.com/en-us/microsoft-365/blog/2019/09/18/why-banks-adopt-modern-cybersecurity-zero-trust-model/)  
High level overview of the advantages in adopting a modern security approach for bankers and their customers.

[ZScaler and MAN](https://www.youtube.com/watch?v=P0XnqGDdXaA&feature=youtu.be)  
MAN energy solutions has implemented Zscaler services as alternative to “traditional” VPN solutions to publish internal applications (to over 5.000 employees).

## Google’s "zero trust" approach with "BeyondCorp"
[Google Online Security Blog: How Google adopted BeyondCorp](https://security.googleblog.com/2019/06/how-google-adopted-beyondcorp.html)  
BeyondCorp is Google’s implementation of the zero trust security model which was also implemented within the company. Learn more about this approach in the following white papers and research papers:

* [BeyondCorp: A New Approach to Enterprise Security](https://ai.google/research/pubs/pub43231)
* [Migrating to BeyondCorp: Maintaining Productivity While Improving Security](https://ai.google/research/pubs/pub46134)
*  [Run Zero Trust Security Like Google](https://www.beyondcorp.com/blog/beyondcorp-weekly-52) 

[BeyondCorp Beyond Google (Cloud Next ’18)](https://www.youtube.com/watch?v=ei1CxF1BHh4)  
Recorded session from Google’s cloud conference (Cloud Next) about the transformation project of Veolia (french company with over 169,000 employees) in 2018.


<span style="color:silver;font-style:italic;font-size:small">Cover image by [bsdrouin from pixabay.com](https://pixabay.com/photos/network-server-system-2402637/)</span>
