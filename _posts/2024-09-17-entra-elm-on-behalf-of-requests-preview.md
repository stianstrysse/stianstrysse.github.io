---
layout: post
title: New feature in Entra ID Governance ELM - support for request on-behalf-of
subtitle: Microsoft has just released a hot, new feature in their cloud IGA offering, Entra ID Governance (Entitlement Management), allowing request on-behalf-of scenarios.
thumbnail-img: /assets/img/posts/2024-09-17/entra-id-governance.png
categories: AZUREAD AZURE IDENTITY ENTRAID ENTRA ELM IDENTITYGOVERNANCE IGA
author: Stian A. Strysse
---

Think about it - if you have access packages governing access to Azure Virtual Desktop, Windows Cloud PC, Citrix, or other remote tools, and new hires need access from day one, managers would historically have to place a manual request to an IGA admin who would then add users to the correct access packages manually. 

Well, not anymore! Say hello to the new 'request access packages on-behalf-of other users' feature now in public preview, available for customers with the `Microsoft Entra ID Governance` or `Microsoft Entra Suite` licenses.

The current preview allows IGA admins to enable the functionality per access package policy, specifically allowing managers to request packages on-behalf-of their direct reports. The default setting is not to allow on-behalf-of requests, which means it can be gradually rolled out on access packages where it makes sense to offer this capability. Once enabled, managers can visit the MyAccess portal, look up the correct access package, select it, and choose which user to request for. The request follows the normal approval configuration, but there's also an option to bypass the approval process if the manager is also the approver of the request - which makes sense in those access packages where managers are specified as approvers.

It's time to start paying more attention to access package requests to spot when the target user is not the same as the requestor.

The public preview currently only supports managers requesting on-behalf-of their direct reports, but I'm really hoping to see this expanded before general availability. Examples of other requestors that could benefit from requesting on-behalf-of other users include application or system owners, helpdesk staff, colleagues helping with onboarding of new team members, etc.

I have to say, Microsoft is really picking up the pace with their cloud IGA offering. Hope to see more cool and highly awaited features soon!

Check out the [official documentation](https://learn.microsoft.com/en-us/entra/id-governance/entitlement-management-request-behalf) and start testing it in your environment today.