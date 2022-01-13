---
layout: post
title: Block legacy authentication protocols using Azure AD Conditional Access policy
subtitle: Let's look at blocking legacy authentication protocols in a global company's Azure AD with full control and ease of mind
categories: AZUREAD AZURE IDENTITY GOVERNANCE CONDITIONAL ACCESS POLICY
thumbnail-img: /assets/img/posts/2022-01-13/aad-disable-legacy-auth-thumbnail.png
author: Stian A. Strysse
---

I recently worked with a global company to help them tighten the security in their Azure AD tenant, including blocking legacy authentication protocols with Conditional Access policies. Now, blocking legacy authentication isn't anything new, and there are [official Microsoft documentation](https://docs.microsoft.com/en-us/azure/active-directory/conditional-access/block-legacy-authentication), guides and blog posts covering this topic, but none the less I wanted to blog about how I implemented this in a controlled rollout.

![AAD Block Legacy Auth thumbnail](/assets/img/posts/2022-01-13/aad-disable-legacy-auth-thumbnail.png)

Blogpost contents:

+ [What is legacy authentication?](#what-is-legacy-authentication)
+ [Why block legacy authentication?](#why-block-legacy-authentication)
+ [How to block legacy authentication?](#how-to-block-legacy-authentication)
    + [Check sign-in logs for any legacy authentication usage](#check-sign-in-logs-for-any-legacy-authentication-usage)
    + [Prepare the Conditional Access policy and test it](#prepare-the-conditional-access-policy-and-test-it)
    + [Enable the policy for all users](#enable-the-policy-for-all-users)
    + [Next steps](#next-steps)

## What is legacy authentication?

Legacy authentication, also referred to as basic auth, means all authentication protocols only supporting a username and a password credential. These protocols do not support modern authentication mechanisms like MFA (multi-factor authentication) and other security capabilities. Legacy authentication protocols are old and not built for today's threat landscape, simply put. 

POP3, IMAP, SMTP, Exchange ActiveSync, Exchange Online Powershell and Exchange Web Services are examples that utilize legacy authentication. A full list of these protocols are listed on [Microsoft Docs](https://docs.microsoft.com/en-us/azure/active-directory/conditional-access/block-legacy-authentication#messaging-protocols-that-support-legacy-authentication). Native mail clients on mobiles (Exchange ActiveSync) and outdated Office apps are known to utilize legacy authentication.

## Why block legacy authentication?

Blocking legacy authentication protocols in Azure AD has been possible for several years using Conditional Access policies, and is highly recommended by Microsoft. The reason is that legacy authentication protocols, as mentioned, do not support modern authentication mechanisms that can fend of attackers. Blocking legacy authentication is very important - even if a tenant requires MFA for all user sign-ins, an attacker would still be able to connect to Exchange Online using basic authentication (username + password) if the password is known or guessed/brute-forced by the attacker, effectively circumventing MFA and other security mechanisms.

Microsoft have [announced](https://techcommunity.microsoft.com/t5/exchange-team-blog/basic-authentication-and-exchange-online-september-2021-update/ba-p/2772210) that they will permanently deactivate all basic authentication protocols in Exchange Online, except for Authenticated SMTP in tenants where this is in use, effective October 1 of this year. This will greatly enhance the security posture of the tenants that still haven't blocked legacy authentication. It will likely also impact services in use by a lot of organizations, so it is time to prepare for this change. If you have services and scripts utilizing POP3, IMAP, the old Exchange Powershell module and so on, you will have to do some changes soon enough.

## How to block legacy authentication?

By using a [Conditional Access policy](https://docs.microsoft.com/en-us/azure/active-directory/conditional-access/overview#:~:text=Conditional%20Access%20policies%20at%20their,factor%20authentication%20to%20access%20it.) we can effectivly block all sign-ins utilizing legacy authentication protocols. Using Conditional Access policies requires Azure AD Premium P1 license. Tenants can also utilize [Azure AD Security Defaults](https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/concept-fundamentals-security-defaults) which requires no licenses, but this option provides no overrides or configuration - meaning that you cannot exclude any users from the policy.

I will now go through the steps for what I did to successfully implement a Conditional Access policy blocking legacy authentication with no negative impact on users or services.

### Check sign-in logs for any legacy authentication usage

The company was already [exporting Azure AD logs to an Azure Log Analytics workspace](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/howto-integrate-activity-logs-with-log-analytics). This means I could utilize one of the built-in Workbooks in Azure AD to start analyzing legacy authentication. In the [Azure AD Portal](https://aad.portal.azure.com), I went to the **Workbooks** blade and opened the `Sign-ins using Legacy Authentication` workbook. This workbook automatically aggregates the sign-in logs streamed to Log Analytics and presents a dashboard.

[![AAD Sign-In Workbook](/assets/img/posts/2022-01-13/aad-workbook-overview.png)](/assets/img/posts/2022-01-13/aad-workbook-overview.png)

The Woorkbook displayed 5 legacy authentication protocols in use during the selected time range. I browsed through these and looked at the users and sign-ins involved. I identified a set of service accounts using POP3, another service account using the legacy Exchange Online Powershell module, a few users utilizing Exchange ActiveSync and AutoDiscover (likely native mobile mail clients), and a huge bulk of "Unknown" - the last one made me a bit uncertain but after checking further it seems that when users in the company authenticates in other Azure AD tenants as B2B guest users, these sign-ins are tagged as "Unknown".

Now I wanted to download the raw sign-in logs to drill down further in the data. I visited the **Sign-in logs** blade in the Azure AD Portal, chose to display data from the last 30 days, and added the following filters:

- **Status:** `Success` (no need to look at failed sign-ins)
- **Client app:** Checked all `Legacy Authentication Clients` (count 13)

I clicked **Apply** and Azure AD filtered the logs according to my configuration.

[![AAD Sign-In Logs LegacyAuth](/assets/img/posts/2022-01-13/aad-signin-logs-legacyauth.png)](/assets/img/posts/2022-01-13/aad-signin-logs-legacyauth.png)

Browsing the logs in the Azure AD Portal wasn't really optimal, so I clicked the **Download** button and downloaded the following logs in JSON format:

- InteractiveSignIns
- NonInteractiveSignIns

With the downloaded sign-in logs on my local disc I wrote the following Powershell script to process the JSON files:

```powershell
# Fetch sign-in logs (JSON)
$InteractiveSignIns = Get-Content -Path ".\InteractiveSignIns_daterange.json" | ConvertFrom-Json
$NonInteractiveSignIns = Get-Content -Path ".\NonInteractiveSignIns_daterange.json" | ConvertFrom-Json

# Process each sign-in record and fetch the necessary data
$report = @(

    # Process InteractiveSignIns log
    $InteractiveSignIns | ForEach-Object {
        $record = $_

        [PSCustomObject]@{
            id = $record.id
            createdDateTime = $record.createdDateTime
            signInEventTypes = $record.signInEventTypes[0]
            isInteractive = $record.isInteractive
            userDisplayName = $record.userDisplayName
            userPrincipalName = $record.userPrincipalName
            userType = $record.userType
            clientAppUsed = $record.clientAppUsed
            appDisplayName = $record.appDisplayName
            userAgent = $record.userAgent
            deviceDetailBrowser = $record.deviceDetail.browser
        }
    }

    # Process NonInteractiveSignIns log
    $NonInteractiveSignIns | ForEach-Object {
        $record = $_

        [PSCustomObject]@{
            id = $record.id
            createdDateTime = $record.createdDateTime
            signInEventTypes = $record.signInEventTypes[0]
            isInteractive = $record.isInteractive
            userDisplayName = $record.userDisplayName
            userPrincipalName = $record.userPrincipalName
            userType = $record.userType
            clientAppUsed = $record.clientAppUsed
            appDisplayName = $record.appDisplayName
            userAgent = $record.userAgent
            deviceDetailBrowser = $record.deviceDetail.browser
        }
    }
)

# Group sign-in logs by UserPrincipalName
$reportGrouped = $report | Group-Object -Property userPrincipalName

# Process grouped sign-in logs and output 1 record per user with all unique legacy auth clients
$reportSorted = $reportGrouped | ForEach-Object {
    $reportGroup = $_
    $reportGroupExpanded = $reportGroup.group
    $reportGroupExpandedSorted = $reportGroupExpanded | Sort-Object -Property clientAppUsed,appDisplayName -Unique

    [PSCustomObject]@{
        userDisplayName = $reportGroupExpandedSorted[0].userDisplayName
        userPrincipalName = $reportGroupExpandedSorted[0].userPrincipalName
        NumOfAuths = $reportGroupExpanded.count
        clientAppUsed = ($reportGroupExpandedSorted.clientAppUsed | Sort-Object -Unique) -join "; "
        appDisplayName = ($reportGroupExpandedSorted.appDisplayName | Sort-Object -Unique) -join "; "
        deviceDetailBrowser = ($reportGroupExpandedSorted.deviceDetailBrowser | Sort-Object -Unique) -join "; "
    }
}

# Output results
$reportSorted | Out-GridView
```

Running the Powershell script provided a report with good overview of the user accounts currently utilizing legacy authentication protocols in the tenant:

[![AAD Sign-In Logs LegacyAuth](/assets/img/posts/2022-01-13/aad-signin-logs-powershell-output.png)](/assets/img/posts/2022-01-13/aad-signin-logs-powershell-output.png)

With this information at hand I contacted the company and talked with them about my findings. 

- A few users utilized non-standard mobile mail clients. The company contacted these users to move them over to Outlook on iOS/Android.
- Several service accounts connected to Exchange Online using POP3. These will be allowed to keep using legacy auth for the time being.
- A couple of service accounts for meeting room equipment connected to Exchange Online using Exchange Web Services.
- A few users utilized legacy authentication with `Windows Azure Active Directory` and `OfficeClientService ldcrl`. When analyzing in detail I found out that some deprecated versions of Microsoft Visio and Project were in use. The company was already in the progress of upgrading to newer versions.

All in all, not too much legacy authentication in use for a company with several thousand employees worldwide.

### Prepare the Conditional Access policy and test it

Microsoft has recently published a great new feature in Azure AD, currently in public preview, which helps out with [creating common Conditional Access Policies from a template](https://docs.microsoft.com/en-us/azure/active-directory/conditional-access/concept-conditional-access-policy-common#:~:text=Conditional%20Access%20templates%20are%20designed,various%20customer%20types%20and%20locations.). 

I visited the Azure AD Portal and clicked **Security** -> **Conditional Access**, then clicked **+ New policy** -> **Create new policy from template**. I then chose the following options:

- **Select a template category:** `Identities`
- **Select template:** `Block legacy authentication`
- **Policy state:** `Off`

Once the Conditional Access policy was deployed I opened it and verified the configuration. I confirmed that the policy will _only_ impact legacy authentication for users in scope, and block matching sign-ins once the policy is enabled:

- **Assignments:** `All users included and specific users excluded` (my own account is excluded)
- **Cloud apps or actions:** `All cloud apps`
- **Conditions:** `1 condition selected` (Client apps: Exchange ActiveSync clients, Other Clients)
- **Grant:** `Block access`

It is always best-practise to exlude at least 1 user account with administrative privilegies, like a [break-the-glass account](https://docs.microsoft.com/en-us/azure/active-directory/roles/security-emergency-access), in case of any issues with a Conditional Access policy. So I left my admin account in the exclusion list of the policy. I also needed to exclude the few service accounts which still needed to access Exchange Online using POP3, until the services can be modernized, and I had the choice of either adding those service accounts to an Azure AD group and then exclude that whole group, or adding the service accounts one-by-one to the policy exclusion list. I went with the latter as I really don't like exclusion groups. In most organizations, many low privilegied administrators are group administrators, so this is always a factor when considering the risk of having an exclusion group which many admins can add user accounts to.

After adding the approved service accounts to the exclusion list, I created a temporary Azure AD group and added it to the inclusion list in the policy. This was because we wanted a group of pilot users to be assigned the policy and work a few days to see if they experienced any issues caused by the policy. Just to be safe, and also to prove to the company that blocking legacy authentication will not impact users or services in a negative way.

After configuring the policy assignments I enabled the policy and waited a few days for any feedback from the pilot users.

### Enable the policy for all users

We agreed on a go-live date after verifying that the pilot users were not impacted negatively by the Conditional Access policy. All users within the company were informed in advance of the implementation, and I had a simple roll-back plan at hand if anything should go awry: _disable the Conditional Access policy_. At the morning of the go-live date I downloaded Azure AD sign-in logs again and verified that no _new_ accounts showed up.

{: .box-note}
**Note**: You can enable a Conditional Access policy in [report-only mode](https://docs.microsoft.com/en-us/azure/active-directory/conditional-access/concept-conditional-access-report-only) to evaluate the impact before enforcing the policy. During sign-in, policies in report-only mode are evaluated but not enforced, and the policy results are logged in the Conditional Access and Report-only tabs of the Sign-in log details. This can help creating confidence in the configuration of a Conditional Access policy prior to enforcing it tenant-wide.

First I verified that the Conditional Access policy exclusion list was correct, then I changed the inclusion list from `Select users and groups` (with the pilot users group) to `All users`. This assigned all users in the tenant to the policy, except for the accounts in the exclusion list. I then enabled the policy.

It was time to start looking at the Azure AD sign-in logs live to catch any issues. I went to the Azure AD Portal and the **Sign-in logs** blade, and configured filters:

- **Status:** `Success` (no need to look at failed sign-ins, as these are stopped prior to Conditional Access policy verification)
- **Client app:** Checked all `Legacy Authentication Clients` (count 13)

This revealed all successfull legacy authentication records, both those that were blocked by the Conditional Access policy, and those that weren't blocked. I checked both **User sign-ins (interactive)** and **User sign-ins (non-interactive)** tabs. I saw a lot of records for the excluded service accounts, and that they were not impacted by the policy at all due to `Conditional Access = Not Applied`. I also saw a few users that was being blocked by the policy due to `Conditional Access = Failure`:

[![AAD Sign-In Logs after CA1](/assets/img/posts/2022-01-13/aad-signin-logs-legacyauth-after-ca.png)](/assets/img/posts/2022-01-13/aad-signin-logs-legacyauth-after-ca.png)

I clicked on one of the sign-in records that was being blocked and went to the **Conditional Access** tab, and confirmed that it was indeed the correct policy that was blocking the sign-in:

[![AAD Sign-In Logs after CA1](/assets/img/posts/2022-01-13/aad-signin-logs-legacyauth-after-ca2.png)](/assets/img/posts/2022-01-13/aad-signin-logs-legacyauth-after-ca2.png)

Clicking **Show details** displayed how the policy matched and that access was correctly blocked. This was a user who was still using Exchange ActiveSync via native mail client on iOS:

[![AAD Sign-In Logs after CA1](/assets/img/posts/2022-01-13/aad-signin-logs-legacyauth-after-ca3.png)](/assets/img/posts/2022-01-13/aad-signin-logs-legacyauth-after-ca3.png)

Everything looked good, the policy was behaving as expected. Now I waited until the next day to get more logs to verify, and then used [Conditional Access policy Insights and Reporting](https://docs.microsoft.com/en-us/azure/active-directory/conditional-access/howto-conditional-access-insights-reporting) workbook to check the overall impact of the policy. I went to the Azure AD Portal -> **Security** -> **Conditional Access** -> **Insights and reporting**, and set the time range in the workbook to last 24 hours and selected the correct Conditional Access policy:

[![AAD CA insights 1](/assets/img/posts/2022-01-13/aad-ca-insight1.png)](/assets/img/posts/2022-01-13/aad-ca-insight1.png)

The dashboard revealed that only 3 users were being blocked by the policy, and that the majority of sign-ins were not affected since the policy does not apply:

[![AAD CA insights 2](/assets/img/posts/2022-01-13/aad-ca-insight2.png)](/assets/img/posts/2022-01-13/aad-ca-insight2.png)

Clicking **Failure** drilled down into these blocked sign-ins, and I confirmed that just a few users were being blocked from signing in to Exchange Online with legacy authentication on their mobile devices, as was expected. These users need to start using the Outlook app, as is the company policy:

[![AAD CA insights 3](/assets/img/posts/2022-01-13/aad-ca-insight3.png)](/assets/img/posts/2022-01-13/aad-ca-insight3.png)

The policy was indeed working as expected, and we now have another layer of defense in place in the tenant to prevent attackers from accessing data via legacy authentication protocols.

### Next steps

The next task is to [disable basic authentication protocols in Exchange Online](https://docs.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/disable-basic-authentication-in-exchange-online?WT.mc_id=Portal-fx) by default and only allow basic authentication for specific accounts by using Exchange Online Authentication policies. Allowing basic auth for a subset of accounts is still a security risk, so it should only be used as a temporary workaround. Implementing Exchange Online Authentication policies will have to go into a later blog post.

As already mentioned, Microsoft have [announced](https://techcommunity.microsoft.com/t5/exchange-team-blog/basic-authentication-and-exchange-online-september-2021-update/ba-p/2772210) that they will permanently deactivate all basic authentication protocols in Exchange Online, except for Authenticated SMTP in tenants where this is in use, effective October 1 of this year. This means we should start immediately to look at the services using the service accounts excempted by the Conditional Access policy, and change them over from legacy authentication to modern authentication. If we do nothing, these services will stop working on October 1. Don't run out of time, good luck!

And that concludes this blogpost, thanks for reading!

Be sure to provide any feedback on [Twitter](https://twitter.com/stianstrysse/status/1478337726948810752) or [LinkedIn](https://www.linkedin.com/posts/stianstrysse_getting-started-with-custom-security-attributes-activity-6884103665645318144-YHC_).