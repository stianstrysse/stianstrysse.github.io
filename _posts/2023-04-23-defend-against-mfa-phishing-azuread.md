---
layout: post
title: Defend against MFA phishing of Azure AD user identities
subtitle: Securing identities in Azure AD with MFA is a no-brainer, but is it really enough? Let's look at attack vectors for the various MFA methods, and how to defend against MFA phishing in Azure AD.
thumbnail-img: /assets/img/posts/2023-04-23/defend-mfa-phishing.jpg
categories: AZUREAD AZURE IDENTITY JIT JEA ZEROTRUST LEASTPRIVILEGE MFA PHISHING CONDITIONALACCESS
author: Stian A. Strysse
---

Continuing on from the [Securing user identities in Azure AD beyond MFA](https://learningbydoing.cloud/blog/securing-user-identities-in-azure-ad-beyond-mfa/) blog post, but this time looking at how to prevent MFA phishing attacks.

Phishing of user account credentials has been part of the cyber threat landscape for ages. This also goes for MFA phishing, or rather phishing attacks where a threat actor gets away with a valid access token, primary refresh token and session cookies for an Azure AD user account.

Now, how is it even possible to fall victim to a phishing attack when the account is protected by MFA?

## Burn the password

Passwords only are bad. Passwords can be intercepted, phished, leaked, guessed, or shared. Single-factor authentication simply is not enough.

## Any MFA method is better than none

Some MFA methods are weak and some are strong. But no MFA is always worse than a weak MFA method. Say all you want about false security - I'd go for the weaker MFA method any day of the week if the only option was to have no MFA.

According to Microsoft, a whooping 99% of compromised user accounts were not protected by MFA.

## Weak MFA methods

Phone-call and SMS, [the weakest form of MFA](https://techcommunity.microsoft.com/t5/microsoft-entra-azure-ad-blog/it-s-time-to-hang-up-on-phone-transports-for-authentication/ba-p/1751752). This might differ from country to country, but mobile subscriptions can be vulnerable to SIM swapping by attackers which fraudulently takes control of a user's mobile subscription by persuading their mobile carrier to transfer the phone number to a SIM card in the attacker's possession. Another possible attack vector is simply put SMS interception - where an attacker catches the SMS contents which is not encrypted.

## Strong MFA methods

[Nudge users](https://learn.microsoft.com/en-us/azure/active-directory/authentication/how-to-mfa-registration-campaign) to move away from weak MFA methods and over to Microsoft's Authenticator app which has has several great security and usability features:

* [Number matching](https://learn.microsoft.com/en-us/azure/active-directory/authentication/how-to-mfa-number-match)
* [Additional context](https://learn.microsoft.com/en-us/azure/active-directory/authentication/how-to-mfa-additional-context)
* [Passwordless sign-in](https://learn.microsoft.com/en-us/azure/active-directory/authentication/howto-authentication-passwordless-phone)
* [GPS location](https://learn.microsoft.com/en-us/azure/active-directory/conditional-access/location-condition)
* Protect app with pin or biometrics
* Onboard Azure AD accounts directly in the app with [Temporary Access Pass](https://learn.microsoft.com/en-us/azure/active-directory/authentication/howto-authentication-temporary-access-pass)
* It's free and supports iOS and Android

Other strong methods are OATH verification code apps like Google Authenticator, and [hardware tokens with OTP](https://learn.microsoft.com/en-us/azure/active-directory/authentication/concept-authentication-oath-tokens). Unfortunately, these are all vulnerable to MFA phishing, including the Microsoft Authenticator app.

## Evilginx attack framework

Evilginx is an open-source framework utilized to conduct  phishing attacks using man-in-the-middle technics to catch credentials, tokens and cookies. Simply explained, an attacker can set up a phishing site looking like a normal Azure AD sign-in page, then lure users to click a link to the site by using e-mail or other means.

If the user actually do sign-in, the phishing website will look and act like the normal sign-in process. But behind the curtains it will proxy the credentials from the attacker's webserver directly to Azure AD, triggering MFA prompt which the user might approve, and if they do approve - the access token, primary refresh token and session cookies are issued to the attacker's server instead of the phished user's device.

## Phishing-resistant MFA methods

This phishing attack works on all types of MFA methods currently supported by Azure AD, except for those methods we call phishing-resistant; Windows Hello for Business, FIDO2 security keys and Certificate-based Authentication (CBA). The reason is that these three authentication methods are securely coupled to Azure AD on a physical device, and cannot be intercepted by an attacker's webserver as the user have no way of providing these credentials to services that are not Azure AD.

If the user had phishing-resistant MFA when signing in to the phishing site, the attack would stop dead in its tracks.

## Obvious solution?

Assign FIDO2 security keys to all users and we're good to go! Well, unfortunately it might not be that easy after all as adoption takes more effort than for other MFA methods. The keys have a cost, it's a physical device and requires some form of management and distribution, and Azure AD demands some other valid MFA method when registering a FIDO2 security key on a user's account. Also, having fallback methods in the case a FIDO2 key is lost might be something to think about.

Windows Hello for Business is a no-brainer, and now it's even easier to implement. WHfB signs the user in with PIN or biometrics, without a password on the user's enrolled Windows device, and it's always regarded as MFA by Azure AD. Say goodbye to MFA prompts and password reset calls.

## Conditional Access policies

Another way of protecting users from MFA phishing without phishing-resistant MFA methods, is by using Conditional Access policies. Conditional Access can require hybrid-joined or compliant device for users signing in, effectively blocking users from signing in from unmanaged devices - like an attacker's phishing website.

If an organization is able to, I would strongly advise to require hybrid-joined or compliant device with CA - as it helps prevent phishing attacks, and also prevents users from connecting to company resources from devices that have an unknown security posture. Windows, MacOS, Linux, iOS and Android are all supported by Intune enrollment. Remember - attackers have gained initial entry to organizations by pivoting from users' home devices utilized to accessing company resources in several publicly known security breaches.

## Secure device enrollment process

Even though Conditional Access policy requires hybrid-joined or compliant device for users, organizations are potentially [vulnerable to fraudulent device registration](https://learn.microsoft.com/en-us/azure/active-directory/standards/memo-22-09-multi-factor-authentication#protection-from-external-phishing) during MFA phishing attacks if users are allowed to enroll devices into Intune. Supporting self-enrollment is of course a great user experience, but it comes with a risk. Allowing an attacker to get a compliant device in Azure AD is definitely very bad.

How about this - only allow Intune enrollment of devices for new hires, so they can get their Windows device installed via Autopilot and their mobile devices enrolled. Remove the access after onboarding has been completed.

To allow enrollment of e.g. new mobile devices in Intune for existing users, set up an access package granting access to this - with or without approval, and with expiration of a few hours.

`Least-privilege` and `just-in-time` principals like this can really save the day when (not if) a user in the organization falls victim to a phishing attack.

That's all for now, thanks for reading!

Be sure to provide any feedback on [Twitter](https://twitter.com/stianstrysse/status/1650199681580822529), [LinkedIn](https://www.linkedin.com/posts/stianstrysse_defend-against-mfa-phishing-of-azure-ad-user-activity-7055963524124016640-TEZL) or [Mastadon](https://infosec.exchange/@stians/110249486902476311).
