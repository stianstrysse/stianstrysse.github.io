---
layout: post
title: What's lurking in your Microsoft Graph app role assignments?
subtitle: Gain insights for security and compliance reasons by reporting on all service principals in Entra ID (former Azure AD) having Microsoft Graph app Role assignments.
thumbnail-img: /assets/img/posts/2023-08-16/audit-graph-app-roles.png
categories: AZUREAD AZURE IDENTITY ENTRAID ENTRA PRIVILEGEDACCESS MICROSOFTGRAPH APPROLE SCOPES
author: Stian A. Strysse
---

Application permissions, often called app role assignments in Entra ID (former Azure AD), are permission sets that an app, service principal or managed identity can be assigned in another resource app, and that app's identity can then access and utilize the resource app's API without a signed-in user present.

Service principals by default have no access to enumerate other objects in Entra ID (former Azure AD). An example, if a service principal used in automated workflows requires read access to all user objects in Entra ID, it will need to be assigned the app role `User.Read.All` [application permission](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-permissions-and-consent#permission-types) (app role) in Microsoft Graph, consented by an admin, to be able to query Graph for any or all users.

The [Microsoft Graph API](https://learningbydoing.cloud/blog/getting-started-with-microsoft-graph-part2/) is covering endpoints for most of Entra ID and Microsoft 365 services, and it has a wide range of app roles for providing specific API access. And let's not forget the old and deprecated Azure AD Graph API which has not yet been fully been sunset. It's critical for compliance and security hygiene of the tenant to audit and monitor app role assignments for these resources especially. Do note that there are a lot of other APIs with their own sets of app roles in Entra ID, some examples below, but for this blog post I will focus on Microsoft Graph and Azure AD Graph.

![entra-id-apis](/assets/img/posts/2023-08-16/entra-id-apis.png)

Some app roles can be abused for privilege escalation all the way up to `Global Admin`, as [Andy Robbins](https://twitter.com/_wald0) (co-creator of BloodHound) points out in [this blog post](https://posts.specterops.io/azure-privilege-escalation-via-azure-api-permissions-abuse-74aee1006f48). A Global Admin can do whatever it wants in Entra ID, and it can also [elevate its access for all Azure subscriptions in the tenant with the flip of a switch](https://learn.microsoft.com/en-us/azure/role-based-access-control/elevate-access-global-admin#elevate-access-for-a-global-administrator). This just proves how critical it is to have full control on any high-privilege app roles.

With these things in mind, let's look at how to extract all app role assignments for Microsoft Graph and Azure AD Graph, including other valuable information for each service principal, using Powershell and the [Graph Powershell SDK v2](https://devblogs.microsoft.com/microsoft365dev/upgrade-to-microsoft-graph-powershell-sdk-v2-now-generally-available/) module.

* [Identify highly privileged app roles](#identify-highly-privileged-app-roles)
* [Required scopes in Graph Powershell SDK](#required-scopes-in-graph-powershell-sdk)
* [Extracting data](#extracting-data)
  * [App roles and assignments](#app-roles-and-assignments)
  * [Owner organizations and sign-in activities](#app-roles-and-assignments)
* [Compiling the report](#compiling-the-report)
* [Full script on GitHub](#full-script-on-github)

## Identify highly privileged app roles

I wish there was a list of all Microsoft Graph app roles with tiering information, identifying how privileged each app role is. Since there are none I've added some of the app roles that I know are highly privileged - they will be specifically flagged as Tier 0 in the report. There are many other privileged app roles, but let's start with these as abusing them can lead to Global Admin access. You can easily add other roles and other tiers, depending on what you want to report on.

```powershell
# The tier 0 app roles below are typically what can be abused to become Global Admin.
# NOTE: Organizations should do their own investigations and include any app roles to regard as sensitive, and which tier to assign them.
$appRoleTiers = @{
    'Application.ReadWrite.All'          = 'Tier 0' # SP can add credentials to other high-privileged apps, and then sign-in as the high-privileged app
    'Application.ReadWrite.OwnedBy'      = 'Tier 0' # SP can take ownership over other high-privileged apps, and then add credentials
    'AppRoleAssignment.ReadWrite.All'    = 'Tier 0' # SP can add any app role assignments to any resource, including MS Graph
    'Directory.ReadWrite.All'            = 'Tier 0' # SP can read and write all objects in the directory, including adding credentials to other high-privileged apps
    'RoleManagement.ReadWrite.Directory' = 'Tier 0' # SP can grant any role to any principal, including Global Admin
}
```

## Required scopes in Graph Powershell SDK

When connecting to Microsoft Graph with the Graph Powershell SDK v2 module, the following delegated scopes are required:

1. `Application.Read.All` (to enumerate service principals)
2. `AuditLog.Read.All` (to pull out service principal sign-in activity)
3. `CrossTenantInformation.ReadBasic.All` (to query app owner tenant information for 3.party apps)

```powershell
# Connect to Microsoft Graph
Connect-MgGraph -Scopes "Application.Read.All","AuditLog.Read.All","CrossTenantInformation.ReadBasic.All"
```

Other than that, no special privileges are necessary for the user account connecting to Microsoft Graph - only read-access is used.

## Extracting data

Now let's start extracting the data we need from Microsoft Graph to create the report.

### App roles and assignments

First we are querying for Microsoft Graph and Azure AD Graph's service principal objects in the tenant - by filtering on their [well-known Application IDs](https://learn.microsoft.com/en-us/troubleshoot/azure/active-directory/verify-first-party-apps-sign-in) (`00000003-0000-0000-c000-000000000000` and `00000002-0000-0000-c000-000000000000`). Once the service principals have been found, we are extracting all app roles and app role assignments, creating [hashtable](https://learn.microsoft.com/en-us/powershell/scripting/learn/deep-dives/everything-about-hashtable?view=powershell-7.3) for quick lookups, and lastly joining the app role assignments for both service principals.

```powershell
# Get Microsoft Graph SPN, appRoles, appRolesAssignedTo and generate hashtable for quick lookups
$servicePrincipalMsGraph = Get-MgServicePrincipal -Filter "AppId eq '00000003-0000-0000-c000-000000000000'"
[array] $msGraphAppRoles = $servicePrincipalMsGraph.AppRoles
[array] $msGraphAppRolesAssignedTo = Get-MgServicePrincipalAppRoleAssignedTo -ServicePrincipalId $servicePrincipalMsGraph.Id -All
$msGraphAppRolesHashTableId = $msGraphAppRoles | Group-Object -Property Id -AsHashTable

# Get Azure AD Graph SPN, appRoles, appRolesAssignedTo and generate hashtable for quick lookups
$servicePrincipalAadGraph = Get-MgServicePrincipal -Filter "AppId eq '00000002-0000-0000-c000-000000000000'"
[array] $aadGraphAppRoles = $servicePrincipalAadGraph.AppRoles
[array] $aadGraphAppRolesAssignedTo = Get-MgServicePrincipalAppRoleAssignedTo -ServicePrincipalId $servicePrincipalAadGraph.Id -All
$aadGraphAppRolesHashTableId = $aadGraphAppRoles | Group-Object -Property Id -AsHashTable

# Join appRolesAssignedTo entries for AAD / MS Graph
$joinedAppRolesAssignedTo = @(
    $msGraphAppRolesAssignedTo
    $aadGraphAppRolesAssignedTo
)
```

We can now process each of the app role assignments in `$joinedAppRolesAssignedTo` to create a report, while enriching the data set even further.

```powershell
# Process each appRolesAssignedTo for AAD / MS Graph
$progressCounter = 0
$cacheAppOwnerOrganizations = @()
$cacheServicePrincipalObjects = @()
$cacheServicePrincipalSigninActivities = @()
$cacheServicePrincipalsWithoutSigninActivities = @()
[array] $msGraphAppRoleAssignedToReport = $joinedAppRolesAssignedTo | ForEach-Object {

    $progressCounter++
    $currentAppRoleAssignedTo = $_
    Write-Host "Processing appRole # $progressCounter of $($joinedAppRolesAssignedTo.count)"

    # Lookup appRole for MS Graph
    $currentAppRole = $msGraphAppRolesHashTableId["$($currentAppRoleAssignedTo.AppRoleId)"]
    if($null -eq $currentAppRole) {
        # Lookup appRole for AAD Graph
        $currentAppRole = $aadGraphAppRolesHashTableId["$($currentAppRoleAssignedTo.AppRoleId)"]
    }
```

### Service principals, owner organizations and sign-in activities

The app role assignments in `$joinedAppRolesAssignedTo` does not contain all the information we need about the assigned service principals. So we will query Graph for the service principal objects, the owner organizations for multi-tenant apps, and sign-in activities. To optimize the script we're utilizing cache and only querying each object one time even tho it has multiple app role assignments.

```powershell
    # Lookup servicePrincipal object - check cache
    $currentServicePrincipalObject = $null
    if($cacheServicePrincipalObjects.Id -contains $currentAppRoleAssignedTo.PrincipalId) {
        $currentServicePrincipalObject = $cacheServicePrincipalObjects | Where-Object { $_.Id -eq $currentAppRoleAssignedTo.PrincipalId }
    } 
    
    else {
        # Retrieve servicePrincipalObject from MS Graph
        $currentServicePrincipalObject = Get-MgServicePrincipal -ServicePrincipalId $currentAppRoleAssignedTo.PrincipalId
        $cacheServicePrincipalObjects += $currentServicePrincipalObject
        Write-Host "Added servicePrincipal object to cache: $($currentServicePrincipalObject.displayName)"
    }
```

Note that looking up the app owner organization (for multi-tenant apps) uses `Invoke-MgGraphRequest` with a URI since I haven't found a cmdlet for this in Graph Powershell SDK yet.

```powershell
    # Lookup app owner organization
    $currentAppOwnerOrgObject = $null
    if($null -ne $currentServicePrincipalObject.AppOwnerOrganizationId) {
        # Check if app owner organization is in cache
        if($cacheAppOwnerOrganizations.tenantId -contains $currentServicePrincipalObject.AppOwnerOrganizationId) {
            $currentAppOwnerOrgObject = $cacheAppOwnerOrganizations | Where-Object { $_.tenantId -eq $currentServicePrincipalObject.AppOwnerOrganizationId }
        } 

        else {
            # Retrieve app owner organization from MS Graph
            $currentAppOwnerOrgObject = Invoke-MgGraphRequest -Method GET -Uri "https://graph.microsoft.com/v1.0/tenantRelationships/findTenantInformationByTenantId(tenantId='$($currentServicePrincipalObject.AppOwnerOrganizationId)')"
            $cacheAppOwnerOrganizations += $currentAppOwnerOrgObject
            Write-Host "Added app owner organization tenant to cache: $($currentAppOwnerOrgObject.displayName)"
        }
    }
```

Let's pull the sign-in activities for the service principals. This gives us information about the most recent service principal sign-in and delegated (user) sign-in, which can help us identify stale apps. Note that this newly released reporting endpoint in Graph is still in beta, which is why we need to utilize the `Get-MgBetaReportServicePrincipalSignInActivity` cmdlet.

```powershell
    # Lookup servicePrincipal sign-in activity if not already in no-signin-activity list
    $currentSpSigninActivity = $null
    if($currentServicePrincipalObject.AppId -notin $cacheServicePrincipalsWithoutSigninActivities) {
        if($cacheServicePrincipalSigninActivities.AppId -contains $currentServicePrincipalObject.AppId) {
            $currentSpSigninActivity = $cacheServicePrincipalSigninActivities | Where-Object { $_.AppId -eq $currentServicePrincipalObject.AppId }
        } 

        else {
            # Retrieve servicePrincipal sign-in activity from MS Graph
            $currentSpSigninActivity = Get-MgBetaReportServicePrincipalSignInActivity -Filter "AppId eq '$($currentServicePrincipalObject.AppId)'"
            
            # If sign-in activity was found, add it to the cache - else add appId to no-signin-activity list
            if($currentSpSigninActivity) {
                $cacheServicePrincipalSigninActivities += $currentSpSigninActivity
                Write-Host "Found servicePrincipal sign-in activity and added it to cache: $($currentServicePrincipalObject.displayName)"
            }

            else {
                $cacheServicePrincipalsWithoutSigninActivities += $currentServicePrincipalObject.AppId
                Write-Host "Did not find servicePrincipal sign-in activity: $($currentServicePrincipalObject.displayName)"
            }
        }
    }
```

## Compiling the report

And finally we can generate a PSCustomObject with the data we need for the report.

```powershell
    # Create reporting object
    [PSCustomObject]@{
        ServicePrincipalDisplayName = $currentServicePrincipalObject.DisplayName
        ServicePrincipalId = $currentServicePrincipalObject.Id
        ServicePrincipalType = $currentServicePrincipalObject.ServicePrincipalType
        ServicePrincipalEnabled = $currentServicePrincipalObject.AccountEnabled
        AppId = $currentServicePrincipalObject.AppId
        AppSignInAudience = $currentServicePrincipalObject.SignInAudience
        AppOwnerOrganizationTenantId = $currentServicePrincipalObject.AppOwnerOrganizationId
        AppOwnerOrganizationTenantName = $currentAppOwnerOrgObject.DisplayName
        AppOwnerOrganizationTenantDomain = $currentAppOwnerOrgObject.DefaultDomainName
        Resource = $currentAppRoleAssignedTo.ResourceDisplayName
        AppRole = $currentAppRole.Value
        AppRoleTier = $appRoleTiers["$($currentAppRole.Value)"]
        AppRoleAssignedDate = $(if($currentAppRoleAssignedTo.CreatedDateTime) {(Get-Date $currentAppRoleAssignedTo.CreatedDateTime -Format 'yyyy-MM-dd')})
        AppRoleName = $currentAppRole.DisplayName
        AppRoleDescription = $currentAppRole.Description
        LastSignInActivity = $currentSpSigninActivity.LastSignInActivity.LastSignInDateTime
        DelegatedClientSignInActivity = $currentSpSigninActivity.DelegatedClientSignInActivity.LastSignInDateTime
        DelegatedResourceSignInActivity = $currentSpSigninActivity.DelegatedResourceSignInActivity.LastSignInDateTime
        ApplicationAuthenticationClientSignInActivity = $currentSpSigninActivity.ApplicationAuthenticationClientSignInActivity.LastSignInDateTime
        ApplicationAuthenticationResourceSignInActivity = $currentSpSigninActivity.ApplicationAuthenticationResourceSignInActivity.LastSignInDateTime
    }
}
```

This will generate a list of all the app role assignments for Microsoft Graph and Azure AD Graph, enriched with additional data for the assigned service principals and any configured tier. Here's an example.

```text
ServicePrincipalDisplayName                     : az-sp-idw-reporter
ServicePrincipalId                              : 17805a37-7b8f-4319-b3d9-36fa7fd037fc
ServicePrincipalType                            : Application
ServicePrincipalEnabled                         : True
AppId                                           : cafd0954-9d88-4e33-b4ab-6e681ba2f4a4
AppSignInAudience                               : AzureADMyOrg
AppOwnerOrganizationTenantId                    : eeb4b582-c6fd-4cc0-b12f-b2604b111a4b
AppOwnerOrganizationTenantName                  : MyTenant
AppOwnerOrganizationTenantDomain                : mytenant.onmicrosoft.com
Resource                                        : Microsoft Graph
AppRole                                         : User.Read.All
AppRoleTier                                     : 
AppRoleAssignedDate                             : 2022-11-24
AppRoleName                                     : Read all users' full profiles
AppRoleDescription                              : Allows the app to read user profiles without a signed in user.
LastSignInActivity                              : 14.08.2023 21:08:25
DelegatedClientSignInActivity                   : 
DelegatedResourceSignInActivity                 : 
ApplicationAuthenticationClientSignInActivity   : 14.08.2023 21:08:25
ApplicationAuthenticationResourceSignInActivity : 
```

## Full script on GitHub

I like to explain how the Powershell scripts I publish works, and this blog post does just that. I have also published [the full Powershell script on GitHub](https://github.com/stianstrysse/powershell-scripts/blob/main/AzureAD-AdminRoles-ReportScript.ps1) so you don't have to copy/paste from this page, feel free to check it out.

Thanks for reading!

Be sure to provide any feedback on [X (former Twitter)](https://twitter.com/stianstrysse/status/1571572424516448256) or [LinkedIn](https://www.linkedin.com/posts/stianstrysse_building-a-comprehensive-report-on-azure-activity-6977338056408219649-KTgg).
