---
layout: post
title: Follow 'just-enough-access' principle by scoping resources during role elevation in Azure PIM
subtitle: Utilizing PIM and having no active privileged Azure RBAC or Azure AD roles by default, always requiring 'just-in-time' elevation, is an important risk-reducing strategy. But scoping down access to only the necessary resources during PIM role elevation, adhering to 'just-enough-access', is also best practise. Let's explore how and why.
thumbnail-img: /assets/img/posts/2023-04-04/pim-jit-jea.png
categories: AZUREAD AZURE IDENTITY JIT JEA ZEROTRUST LEASTPRIVILEGE
author: Stian A. Strysse
---

Privileged Identity Management (PIM) in Azure is a service that helps organizations to manage, govern and monitor access to resources in Azure and Azure AD. It helps with reducing risk and exposure by adhering to the principle of `least-privilege` - by providing capabilities for granting privileged access roles to the required resources for the correct individuals at the right time.

One of the principles in `zero-trust` strategy is specifically `least-privilege`, which comprises of `just-in-time` and `just-enough-access` plus other strategies.

![Zero trust principles - by Microsoft (https://learn.microsoft.com/en-us/microsoftteams/shared-device-security-for-microsoft-teams)](/assets/img/posts/2023-04-04/zerotrustprinciples.png)

* [Just-in-time (JIT)](#just-in-time-jit)
* [Just-enough-access (JEA)](#just-enough-access-jea)
* [PIM to the rescue](#pim-to-the-rescue)
* [Activate eligible role for specific scope in PIM](#activate-eligible-role-for-specific-scope-in-pim)
* [Activate eligible role from a resource's 'Access control (IAM)' blade](#activate-eligible-role-from-a-resources-access-control-iam-blade)

## Just-in-time (JIT)

One of the key features of PIM is `just-in-time` (JIT) access, which allows users to gain temporary access to a privileged role or resource for a limited amount of time. JIT access minimizes the exposure of sensitive resources and reduces the attack surface by limiting the time window during which a user can access a privileged role or resource.

An example of JIT is requiring developers to elevate into the `Key Vault Secrets Officer` role when they need to work with secrets, expiring that access within `X` hours - instead of having standing and active access at persistently.

## Just-enough-access (JEA)

JEA, `just-enough-access`, simply put means having the lowest administrative privileges possible, and access only to resources that are strictly necessary to complete a task. Good for security, but at the same time it's lowering operational risk as there's less chance for accidental or intentional  changes to be applied to the wrong resources.

Azure AD supports JEA in certain scenarios with the concept of [Administrative Units](https://learn.microsoft.com/en-us/azure/active-directory/roles/administrative-units). Azure RBAC also supports JEA, as granular and specific roles can be granted all the way down to a single resource in Azure. The challenge is to not give too broad permissions, especially ones that are not necessary 99% of the time. Granting `Contributor` role on the tenant root management group is far from JEA, the blast radius is potentially extreme, but may still be required in certain circumstances.

## PIM to the rescue

PIM can help out with JEA - while an eligible Azure RBAC role has been granted high up in the hierarchy, the role holder can choose which descendant scope to activate it for. If we're so "lucky" to have eligible `Contributor` role on the tenant root management group, effectively granting contributor permissions on all descendent objects once activated, it doesn't mean we have to activate the role with the full scope on every occasion.

Alternatively we can select a specific descendent management group, subscription or resource group we want to activate the contributor role for as scope in PIM. Which means we can choose our `Contributor` role on the tenant root management group, but effectively scope it down to a specific descendant resource group. By doing so we're adhering to the `least-privilege` and `just-enough-access` principles by only elevating for what we actually need then and there.

Yes, we might have to "PIM ourselves" a few times more than only once a day, but really - that's the only drawback. Have to get ourselves a few cups of coffee during the day anyways. While the big advantage is smaller blast radius should something disastrous happen. Think about that for a second.

## Activate eligible role for specific scope in PIM

Start by accessing the PIM portal at [https://aka.ms/pim](https://aka.ms/pim):

1. Select the `Azure resources` blade, click `Activate` on the role we want to elevate into.
2. Choose duration, type a reason and go to the `Scope` blade.
3. In the `Scope` blade, click `Select scope`, choose `Management group`, `Subscription` or `Resource group`.
4. Search for and select the correct resource to scope the role elevation for, then click `Activate`.

We've now elevated our `Contributor` role only for the `p-plt-idm` resource group, instead of for the whole tenant root management group.

![PIM - elevate into role on scope](/assets/img/posts/2023-04-04/pim-scope-elevation.png)

We can also do this programatically, but that'll be saved for another blog post.

## Activate eligible role from a resource's 'Access control (IAM)' blade

This feature is brand new, released in March 2023 according to [Microsoft's documentation](https://learn.microsoft.com/en-us/azure/active-directory/privileged-identity-management/pim-resource-roles-activate-your-roles).

> As of March 2023, you may now activate your assignments and view your access directly from blades outside of PIM in the Azure portal. Read more [here](https://learn.microsoft.com/en-us/azure/active-directory/privileged-identity-management/pim-resource-roles-activate-your-roles#activate-with-azure-portal). Privileged Identity Management role activation has been integrated into the Billing and Access Control (AD) extensions within the Azure portal. Shortcuts to Subscriptions (billing) and Access Control (AD) allow you to activate PIM roles directly from these blades.

1. Go to the `Access control (IAM)` blade of a resource.
2. Click `View my access` and go to the `Eligible assignments` tab.
3. Select the role we want to elevate into, then click `Activate role`.
4. Choose duration, type a reason and go to the `Scope` blade.
5. In the `Scope` blade, click `Select scope`, choose the correct resource to scope the role elevation for and then click `Activate`.

![Access blade - elevate into role on scope](/assets/img/posts/2023-04-04/accessblade-scope-elevation.png)

> Note: This specific functionality seems buggy as of April 4, and I have not been successfull at selecting a descendant resource as scope yet. Microsoft will likely fix this very soon.

This makes it so much easier to elevate into a role for the scope of a resource we're currently working on, and we don't even have to visit the PIM portal. Nice!

That's it for now, thanks for reading!

Be sure to provide any feedback on [Twitter](https://twitter.com/stianstrysse/status/1640426366603542545), [LinkedIn](https://www.linkedin.com/posts/stianstrysse_securing-user-identities-in-azure-ad-beyond-activity-7046191571314032641-EZ8T) or [Mastadon](https://infosec.exchange/@stians/110096763996667030).
