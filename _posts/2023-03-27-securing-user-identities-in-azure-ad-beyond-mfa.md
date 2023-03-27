---
layout: post
title: Securing user identities in Azure AD beyond MFA
subtitle: In today's threat landscape it's important to be healthy paranoid and always re-think how attackers could potentially breach an organization's defenses. So let's explore how to protect user identities in Azure AD beyond MFA
thumbnail-img: /assets/img/posts/2023-03-27/beyond-mfa.png
categories: AZUREAD AZURE IDENTITY MFA ZEROTRUST FIDO2
author: Stian A. Strysse
---

No pretty screenshots this time, but I'll try to keep it short and to the point.

* [The basics](#the-basics)
* [Securing the MFA registration process](#securing-the-mfa-registration-process)
* [No network exceptions](#no-network-exceptions)
* [Block risky identities](#block-risky-identities)
* [Block unknown devices](#block-unknown-devices)
* [Tweak sign-in frequency and browser session persistence](#tweak-sign-in-frequency-and-browser-session-persistence)
* [Final thoughts](#final-thoughts)

## The basics

First - if an organization's MFA coverage for admin and user identities is below 100%, that's where to start. All identities used by individuals with a pulse must be secured using MFA - no exceptions. Only requiring passwords in Azure AD really isn't enough and will eventually cause a disaster.

## Securing the MFA registration process

An essential step is to secure the MFA registration process. Users by default have the ability to register MFA methods upon sign-in when triggering MFA requirement - if no current MFA method exists yet for the account. An attacker could potentially sign-in with compromised credentials and enroll MFA right in the Authenticator app - which would definitely be bad.

To prevent this, [use Conditional Access policy to secure the MFA registration process](https://learn.microsoft.com/en-us/azure/active-directory/conditional-access/howto-conditional-access-policy-registration) by either requiring known/compliant device, or another MFA method like [Temporary Access Pass](https://learn.microsoft.com/en-us/azure/active-directory/authentication/howto-authentication-temporary-access-pass).

## No network exceptions

As we keep focusing on Azure AD, require MFA always and from anywhere. Stop excluding MFA from known and "trusted" office networks. Seriously, the internal networks can't be trusted anymore, it's 2023 and we need to assume breach, never trust and always explicitly verify! If an attacker already is on the inside, excluding MFA from that network is just helping them potentially compromise the organization's cloud resources too.

From Azure AD's perspective, all networks should be treated as the rest of the Internet. Full zero trust.

## Block risky identities

Protect Azure AD user accounts further with Identity Protection signals and Conditional Access policies. Block `high-risk` sign-ins and users - especially if a SOC is present and can act on such events to investigate and quickly remediate both for the sake of security and user productivity.

`Medium-risk` sign-ins could [trigger full re-authentication with MFA](https://techcommunity.microsoft.com/t5/microsoft-entra-azure-ad-blog/new-require-reauthentication-for-intune-enrollment-or-risk/ba-p/3299049), and `medium-risk` users could trigger full re-authentication with secure password-change through [self-service password reset](https://learn.microsoft.com/en-us/azure/active-directory/authentication/tutorial-enable-sspr). This would assist users in helping themselves to get out of a potentially bad situation.

## Block unknown devices

Going further it's important to bring in device status. Which device is the user logging in from, is it known to Azure AD either as an `Azure AD-joined` or `registered` device, or as a `hybrid-joined` device for those organizations with AD domain-joined computers. In case of Azure AD-joined or registered device - is it in compliance with the implemented security policies in Intune?

Using Conditional Access policies to block sign-ins from `unknown` or `non-compliant` devices might be the protection mechanism fending off an actual ongoing phishing attack at the very last minute. As long as users are allowed to utilize phishable MFA methods like SMS, phone call, OTP and even Microsoft Authenticator - they are at risk to be lured into giving up both their account credentials and MFA challenge on a phishing site running [Evilginx2](https://janbakker.tech/how-to-set-up-evilginx-to-phish-office-365-credentials/) (as showcased by [@janbakker_](https://twitter.com/janbakker_)) or similar tool.

With this Conditional Access policy in place the attacker would be stopped dead in its tracks when signing in with the phished credentials, as the `known device` requirement can't be satisfied.

## Tweak sign-in frequency and browser session persistence

Deviation from the default sign-in frequency for standard users, which is a rolling window of 90 days, is likely not a good strategy for managed devices in the organization. It may impact user productivity, and will likely annoy users more than securing them.

However, in certain scenarios for specific `personas` it makes sense to both configure sign-in frequency to a day or less - and to disallow persistent browser session in one go:

* Highly privileged user accounts.
* Users accessing apps from unmanaged devices.

Make sure to understand how these features work by looking at [Microsoft's documentation](https://learn.microsoft.com/en-us/azure/active-directory/conditional-access/howto-conditional-access-session-lifetime).

## Implement phishing-resistant MFA

Lastly, look into transitioning over to phishing-resistant MFA methods. High-value targets like privileged users and VIPs should be required to use FIDO2 security keys, but it's even more important to require FIDO2 for any user excluded in the `known device` Conditional Access policy - as they are much more susceptible at being successfully compromised in a phishing attack.

## Final thoughts

There are of course many other measures to consider and features to implement for protecting user identities. Some important things comes to mind, especially within endpoint management, like [prevent giving users local admin privileges](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/implementing-least-privilege-administrative-models#on-workstations) by default on their computers and instead look into [Endpoint Privilege Management](https://techcommunity.microsoft.com/t5/microsoft-intune-blog/enable-windows-standard-users-with-endpoint-privilege-management/ba-p/3755710), make sure to use all the goodies in [Windows like Defender for Endpoint](https://learn.microsoft.com/en-us/microsoft-365/security/defender-endpoint/microsoft-defender-endpoint) and [Credential Guard](https://learn.microsoft.com/en-us/windows/security/identity-protection/credential-guard/credential-guard), look into issuing privileged and [secure access workstation](https://learn.microsoft.com/en-us/security/privileged-access-workstations/privileged-access-devices) for high-impact users, and more.

While we're talking about Conditional Access policies, I highly recommend looking into [Microsoft's articles on Conditional Access for zero trust](https://learn.microsoft.com/en-us/azure/architecture/guide/security/conditional-access-zero-trust) - and also [this GitHub repo](https://github.com/microsoft/ConditionalAccessforZeroTrustResources) by [@claus_jespersen](https://twitter.com/claus_jespersen).

That's it for now, thanks for reading - and keep on securing those identities!

Be sure to provide any feedback on [Twitter](https://twitter.com/stianstrysse/status/1640426366603542545), [LinkedIn](https://www.linkedin.com/posts/stianstrysse_securing-user-identities-in-azure-ad-beyond-activity-7046191571314032641-EZ8T) or [Mastadon](https://infosec.exchange/@stians/110096763996667030).
