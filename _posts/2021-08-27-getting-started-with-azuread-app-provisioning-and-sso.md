---
layout: post
title: "Getting started with Azure AD App Provisioning and SSO"
subtitle: "My first-ever blog post walks through how to enable automated user provisioning and single-sign-on from Azure AD to SaaS apps, specifically for Workplace from Facebook"
categories: AZUREAD IDENTITY PROVISIONING SSO
thumbnail-img: "/assets/img/posts/2021-08-27/aad-sso-provisioning-thumb.png"
---

Azure AD has many key strengths and features, automated user provisioning and single-sign-on (SSO) capabilities for a number of SaaS apps being two of them. 

![AAD SSO and Provisioning](/assets/img/posts/2021-08-27/aad-sso-provisioning.png)

I've specifically worked in consultancy focusing on identity, access and security the last five years - and I must honestly say that there's nothing as cool as automating and controlling identity lifecycle and governance with modern and built-in capabilities like the ones Azure AD is offering. 

Having users (and groups) automatically created, maintained and disabled/removed in connected SaaS apps gives organizations full control of the identities, while enabling single-sign-on for the same apps makes it painless for end-users to sign in, and lowers the volume of Service Desk tickets categorized as sign-in issues. Win-win!

In the Enterprise Gallery you will, at the time of writing, find 230 SaaS apps supporting automated user provisioning, and 1675 SaaS apps supporting SSO (SAML or OpenID Connect). Let's take one of those SaaS apps, "Workplace by Facebook" which many organizations use today, and see how we can enable both of these features. 

![AAD Gallery](/assets/img/posts/2021-08-27/aad-gallery.png)

## Prerequisites

> **Note:** As always, test well and get to know the features in a demo environment before planning for production!

I have a demo environment with Azure AD and M365 E5 licenses ready to go, but the Free Tier of Azure AD is sufficient for automated user provisioning to SaaS apps. Note that automated group provisioning requires [Azure AD P1 licenses](https://www.microsoft.com/en-us/security/business/identity-access-management/azure-ad-pricing). 

You need to be either a Global Administrator, Application Administrator or Cloud Application Administrator in the Azure AD tenant for adding the Workplace app and enabling the provisioning and SSO features.

Go to the [Azure AD Gallery](https://aad.portal.azure.com/#blade/Microsoft_AAD_IAM/AppGalleryBladeV2), search for "Workplace from Facebook" and create the app in Azure AD.

![Add Azure AD Gallery App: Workplace from Facebook](/assets/img/posts/2021-08-27/fb-app-add.png)

[Signing up for a "Workplace by Facebook" trial](https://work.workplace.com/company_creation/) grants 30 days of core capabilities, which is required both for automated user provisioning and for single-sign-on to Workplace. Be sure to sign up with a user where the email/UPN is for the same domain as in your Azure AD tenant, as this domain will have to be verified in Workplace before enabling SSO. 

Once signed up with a trial, we can continue with enabling the features.

## Enabling single-sign-on

Enabling single-sign-on in Azure AD for "Workplace by Facebook" is [documented by Microsoft](https://docs.microsoft.com/en-us/azure/active-directory/saas-apps/workplacebyfacebook-tutorial), but I'll go through the steps here. 

1. Navigate to the [Azure Portal](https://portal.azure.com) -> *Enterprise Applications* -> *Workplace by Facebook* (which you added earlier) -> *Properties*, set **User assignment required** to **Yes** and save. 

    Go to *Users and groups* and assign your admin account (the one you registered the Workplace environment with), as this account is needed for testing SSO before being allowed to save the SSO settings in Workplace.

2. In a separate browser tab, navigate to the [Workplace by Facebook website](https://workplace.com) and log in to your new trial environment, then go to *Admin Panel* -> *Security* -> *Authentication*.

    Activate the **Single-sign on (SSO)** option listed under **Log in** and set **Default for new users** to **SSO**.
    
    Now click the **+ Add new SSO provider** button and copy the following values to notepad:

    | **Setting** | **Example value** |
    | Audience URL | https://www.workplace.com/company/388226916265438 |
    | Recipient URL | https://vanitydomaincloud394.workplace.com/work/saml.php |
    | ACS (Assertion Consumer Service) URL | https://vanitydomaincloud394.workplace.com/work/saml.php |  

    > **Note:** Do not use the example values above, make sure to fetch the correct values from your Workplace environment!

3. Go to the Azure Portal browser tab, navigate to **Single sign-on** in the **Workplace by Facebook** Enterprise App and select **SAML**

    Click **Edit** on **Basic SAML Configuration** and paste in the values you copied to notepad in step 2:

    | **Setting** | **Value** |
    | Identifier (Entity ID) | Set value from: Audience URL |
    | Reply URL (Assertion Consumer Service URL) | Set value from: ACS (Assertion Consumer Service) URL |
    | Sign on URL | Set value from: Recipient URL |  

    > **Note:** Set the new value in 'Identifier (Entity ID)' as default, and remove the 'https://www.facebook.com/company/*' value from the list before clicking **Save**!

    ![AAD SAML configuration](/assets/img/posts/2021-08-27/aad-saml-config.png)

    Then click **Save** and close the **Basic SAML Configuration** - choose **No, I'll test later** if you're asked to test single-sign-on

4. In the Azure Portal browser tab, under **SAML Signing Certificate**, click **Download** on **Certificate (Base64)**, and open the downloaded *Workplace from Facebook.cer* file in notepad to view the base64 value.

    Under **Set up Workplace from Facebook** on the same page, click the **Configuration URLs** dropdown and copy the following values to notepad:

    | **Setting** | **Example value** |
    | Login URL | https://login.microsoftonline.com/56aa0fa5-b7dd-4dbc-81da-3d57b2d69fff/saml2 |
    | Azure AD Identifier | https://sts.windows.net/56aa0fa5-b7dd-4dbc-81da-3d57b2d69fff/ |
    | Logout URL | https://login.microsoftonline.com/56aa0fa5-b7dd-4dbc-81da-3d57b2d69fff/saml2 |  

    > **Note:** Do not use the example values above, make sure to fetch the correct values from your Azure AD app!

5. In the Workplace website browser tab, paste in the values you copied to notepad in step 4:

    | **Setting** | **Value** |
    | Name of the SSO Provider | Set value as: Azureadsso |
    | SAML Issuer URL | Set value from: Azure AD Identifier |
    | SAML Certificate | Set base64 value from: downloaded certificate |  

    ![FB SSO configuration](/assets/img/posts/2021-08-27/fb-sso-config.png)

    Once the values have been added, click the **Test SSO** button. If everything has been done correctly, you should see a new tab stating *Your SSO is working properly*

    ![FB SSO working](/assets/img/posts/2021-08-27/fb-sso-working.png)

    Now click **Save Changes** on the **Single Sign-On (SSO) Setup** dialog box in the Workplace website. 
    
6. Still in the Workplace website browser tab, go to **Admin Panel** -> **Settings** -> **Email Domains** and follow the instructions to verify the email domain. 

    Once verified, go back to **Admin Panel** -> **Security** -> **Authentication**, click the **Assign Email Domains** button and add your verified domain - click **Save**. Also verify that **Allow people to authenticate using** is set to **Single-sign on (SSO)** and **Default for new users** is set to **SSO**
    
    > **Note:** Remember to click **Save Changes** on the **Authentication** page once everything is configured.

    ![FB SSO configured](/assets/img/posts/2021-08-27/fb-sso-completed.png)

This concludes the setup of single-sign-on. Any Azure AD users assigned to the **Workplace from Facebook** app, which also exists as a user in Workplace, will now be able to use single-sign-on.

## Enabling automated user provisioning

Placeholder to do a new build!