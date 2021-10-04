---
layout: post
title: Getting started with Microsoft Graph - part 4
subtitle: The forth blogpost in the series explains the Graph Powershell SDK.
categories: AZUREAD OFFICE365 GRAPH POWERSHELL
share-img: /assets/img/posts/2021-10-04/microsoft-graph-thumb.png
author: Stian A. Strysse
date: 2021-10-04 08:00:00
---

Let's continue with this blogpost series by looking at Microsoft Graph Powershell SDK!

*[Part 1](/blog/getting-started-with-microsoft-graph/)* --- *[Part 2](/blog/getting-started-with-microsoft-graph-part2/)* --- *[Part 3](/blog/getting-started-with-microsoft-graph-part3/)* --- **Part 4**

+ [What is Microsoft Graph PowerShell SDK?](#what-is-microsoft-graph-powershell-sdk)
    + [SDK installation](#sdk-installation)
    + [SDK API version](#sdk-api-version)
    + [SDK authentication](#sdk-authentication)
    + [Working with the Powershell SDK](#working-with-the-powershell-sdk)
    + [SDK examples for user objects](#sdk-examples-for-user-objects)
    + [SDK examples for group objects](#sdk-examples-for-group-objects)
    
_

## What is Microsoft Graph PowerShell SDK?

The Microsoft Graph Powershell SDK is a collection of Powershell modules that contain cmdlets to work with resources in Microsoft Graph. It uses 'Microsoft Authentication Library' (MSAL) instead of the old 'Azure AD Authentication Library' (ADAL) which will be deprecated in 2022. The Microsoft Graph Powershell SDK will replace the old MSOL (MSOnline) and AzureAD Powershell module in the near future, so be sure to get familiar with it now.

### SDK installation

Before installing the Microsoft Graph Powershell SDK, there are some prerequisites that must be met:

- Powershell version 5.1 or later (version 7.x is recommended)
- [.NET Framework version 4.7.2 or later](https://docs.microsoft.com/en-us/dotnet/framework/install/)
- Update PowershellGet to the latest version using the following cmdlet in Powershell: `Install-Module PowerShellGet -Force`

To install the Powershell SDK, run the following cmdlet in Powershell: `Install-Module Microsoft.Graph -Scope CurrentUser`. The full SDK contains a whooping 38 sub modules, so it will take some time to install them.

Since the Powershell SDK is work-in-progress, make sure to update it from time to time with the cmdlet: `Update-Module Microsoft.Graph`

### SDK API version

By default, the Powershell SDK uses the `v1.0` API version. You can easily switch to the `beta` API version with the following cmdlet: 

```powershell
Select-MgProfile -Name "beta"
```

### SDK authentication

The PowerShell SDK supports two types of authentication: delegated access, and [app-only access](https://docs.microsoft.com/en-us/graph/powershell/app-only?tabs=azure-portal). Delegated access is what you will be using when running the Powershell SDK as a user as it invokes operations on your behalf with your privileges, while app-only access is normally used with automation tasks.

When signing in with the Powershell SDK you need to define the permission scopes you want to use. You will need to sign in with an admin user in the tenant to consent to the permission scopes. After authentication, you can add additional required permission scopes by repeating the `Connect-MgGraph` cmdlet with the new permission scopes.

```powershell
Connect-MgGraph -Scopes "User.ReadWrite.All","Group.ReadWrite.All"
```

When you run this cmdlet in Powershell, your most recent Internet browser window will pop up asking you to pick an account to sign in with. Be sure to sign in with the correct account in the correct tenant!

![Graph PS SDK auth 1](/assets/img/posts/2021-10-04/graph-ps-sdk-auth1.png)

Next, you will be asked to do an admin consent to the requested delegated permission scopes (*User.ReadWrite.All* and *Group.ReadWrite.All* in this example). If you choose to *Consent on behalf of your organization*, the next user who authenticates with the Powershell SDK will not be asked to consent.

![Graph PS SDK auth 2](/assets/img/posts/2021-10-04/graph-ps-sdk-auth2.png)

{: .box-note}
**Note**: Any permission scopes you consent to for Graph Powershell SDK is only [delegated permissions type](/blog/getting-started-with-microsoft-graph-part2/#permission-scopes-and-consent), which means that Graph Explorer will never have more privilegies than the siged-in user, and the signed-in user must be present. 

Once consented, the Powershell SDK is authenticated and you can start using it to interact with resources in Azure AD and Microsoft 365. The Powershell SDK will automatically refresh the access token as long as the session is active. 

### Working with the Powershell SDK

All nouns in cmdlets in the Powershell SDK is prefixed with `*-Mg`, like `Get-MgUser`, `New-MgGroup` etc. The documentation for the Powershell SDK is a bit scarse at the moment, so use Powershell functionality to lookup cmdlets.

```powershell
Get-Command *mggroup*
Get-Command *mguser*
Get-Command -Module Microsoft.Graph* *team*
Get-Help New-MgUser -Detailed
Get-Help Remove-MgGroup -Detailed
```

An important cmdlet to know in the Powershell SDK is `Invoke-MgGraphRequest`. With this cmdlet you can do all Graph requests without using a specific `*-Mg*` cmdlet, quite the same as when using Graph Explorer.

As an example, instead of running the cmdlet `Update-MgUser -UserId "AdeleV@M365x955081.OnMicrosoft.com" -JobTitle "New Title"` to update a user's title, you can use:
```powershell
$body = @{
    jobTitle = "New Title"
} | ConvertTo-Json

Invoke-MgGraphRequest -Method PATCH -Uri "https://graph.microsoft.com/v1.0/users/AdeleV@M365x955081.OnMicrosoft.com" -Body $body -ContentType "application/json; charset=utf-8"
```

This means that you can easily test out all the [Graph Explorer examples](/blog/getting-started-with-microsoft-graph-part3/#request-examples-for-user-objects) in this blog post series using the Powershell SDK. I have also found `Invoke-MgGraphRequest` as a lifesaver when I recently ran into a bugged `*-Mg*` cmdlet.

{: .box-warning}
**Warning**: As some of these operations will do actual changes in your tenant, be sure to use an [Azure AD testing environment](/blog/getting-started-with-microsoft-graph-part3/#azure-ad-testing-environment) when trying out stuff!

### SDK examples for user objects

- **Fetch selected attributes on a specific user object and expand reference attribute**

    Use the cmdlet `Get-MgUser` with the `-Property` parameter to specify attributes to retrieve, add the correct UPN or objectId to the `UserId` parameter. Use the `-Expand manager` parameter to retrieve attributes for the user's manager.

    ```powershell
    $user = Get-MgUser -UserId "adelev@m365x955081.onmicrosoft.com" -Property id,displayName,givenName,surname,mail,userPrincipalName,userType -ExpandProperty manager

    # Output user attributes
    $user | Format-List

    # Output manager attributes
    $user.manager.AdditionalProperties
    ```

    *Permission scopes required: User.Read.All*

- **Fetch the first x number of users**

    Use the cmdlet `Get-MgUser` and utilize the `-Top` parameter to specify the maximum number of objects to return. Note that 999 is max, and the default is 100 if `$top` is not specified.

    ```powershell
    Get-MgUser -Top 5
    ```

    *Permission scopes required: User.Read.All*

- **Fetch user(s) with a specific attribute value**

    Use the cmdlet `Get-MgUser` and utilize the `-Filter` parameter to filter the response to only return objects with a specific attribute value.

    ```powershell
    Get-MgUser -Filter "displayName eq 'Adele Vance'"
    Get-MgUser -Filter "accountEnabled eq true"
    ```

    *Permission scopes required: User.Read.All*

- **Fetch users and count the number of returned objects**

    Use the cmdlet `Get-MgUser` and utilize the `-CountVariable` parameter to retrieve a count of returned objects. Note that the parameter `-ConsistencyLevel` with value `eventual` is required for this operation.

    ```powershell
    Get-MgUser -Filter "accountEnabled eq true" -CountVariable mgcounter -ConsistencyLevel eventual

    # Output counter
    $mgcounter
    ```

    *Permission scopes required: User.Read.All*

- **Fetch users created within a specific time period**

    Use the cmdlet `Get-MgUser` and utilize the `-Filter` parameter with dates to specify time periods to filter the response on. Note that the parameter `-ConsistencyLevel` with value `eventual` and `-CountVariable` parameter is required for this operation, as is true for all [advanced Graph queries](https://docs.microsoft.com/en-us/graph/aad-advanced-queries) using `ge` (greater than), `le` (less than) and certain other specific operators.

    ```powershell
    Get-MgUser -Filter "createdDateTime ge 2021-09-17T07:00:00Z and createdDateTime le 2021-09-18T07:00:00Z" -CountVariable mgcounter2 -ConsistencyLevel eventual

    # Output counter
    $mgcounter2
    ```

    *Permission scopes required: User.Read.All*

- **Create a new user**

    Use the cmdlet `New-MgUser` to create a new user account. The attribute parameters in the example below are the very minimum of required attributes, add additional attributes as needed. To generate a strong password, utilize [this powershell script](https://github.com/stianstrysse/powershell-scripts/blob/main/StrongPasswordGenerator.ps1).

    ```powershell
    $passwordProfile = @{
        forceChangePasswordNextSignIn = $true
        password = "HSakz2Gj4YlBA*:6ELtqTcC@i"
    } | ConvertTo-Json

    New-MgUser -DisplayName "Graph Demo User 2" -MailNickname "GraphDemoUser2" -UserPrincipalName "graphdemouser2@M365x955081.onmicrosoft.com" -PasswordProfile $passwordProfile -AccountEnabled:$false
    ```

    *Permission scopes required: User.ReadWrite.All*

- **Update an existing user**

    Use the cmdlet `Update-MgUser` to change data on an existing user account, add the correct UPN or objectId to the `-UserId` parameter. Only include attributes that you want to change in the in attribute parameters.

    ```powershell
    Update-MgUser -UserId "graphdemouser2@M365x955081.onmicrosoft.com" -AccountEnabled:$true -JobTitle "Test Account"
    ```

    *Permission scopes required: User.ReadWrite.All*

- **Delete an existing user**

    Use the cmdlet `Remove-MgUser` to delete an existing user account, add the correct UPN or objectId to the `-UserId` parameter. The account will be soft-deleted for 30 days before Azure AD automatically deletes it permanently.

    ```powershell
    Remove-MgUser -UserId "graphdemouser2@M365x955081.onmicrosoft.com"
    ```

    *Permission scopes required: User.ReadWrite.All*

- **Restore a deleted user**

    Use the cmdlet `Restore-MgUser` to restore a soft-deleted user, add the correct objectId to the `-UserId` parameter.

    ```powershell
    Restore-MgUser -UserId f7b0b597-965f-4fcd-beb0-324335223c5c
    ```

    *Permission scopes required: User.ReadWrite.All*

### SDK examples for group objects

- **Fetch and expand members on a group object**

    Use the cmdlet `Get-MgGroup` and utilize the `-Expand members` parameter to include data on group members in the response, add the correct objectId to the `-GroupId` parameter.

    ```powershell
    $group = Get-MgGroup -GroupId 7991df10-5597-4d63-9040-b66659b6629d -ExpandProperty members

    # Output members
    $group.members

    # Output attributes on first group member
    $group.members[0].AdditionalProperties
    ```

    *Permission scopes required: Group.Read.All*

- **Add a user as a group member**

    Use the cmdlet `New-MgGroupMember`, add the objectId of the group in the `-GroupId` parameter and add the objectId of the user in the `-DirectoryObjectId` parameter to add the user as a member in the specified group.

    ```powershell
    New-MgGroupMember -GroupId 2c16bbc9-7489-44a5-a9b7-67d5d8d8d4b5 -DirectoryObjectId f7b0b597-965f-4fcd-beb0-324335223c5c
    ```

    *Permission scopes required: GroupMember.ReadWrite.All or Group.ReadWrite.All*

- **Create a new dynamic Azure AD security group**

    Use the cmdlet `New-MgGroup` to create a new security group with dynamic membership (`-GroupTypes DynamicMembership`). The attribute parameters in the example below are the very minimum of required attributes, add additional attributes as needed.

    ```powershell
    New-MgGroup -GroupTypes DynamicMembership -DisplayName "Graph Group Dynamic 5" -MailNickname "GraphGroupDynamic5" -MailEnabled:$false -SecurityEnabled:$true -MembershipRule "user.accountEnabled -eq true" -MembershipRuleProcessingState "On"
    ```

    *Permission scopes required: Group.ReadWrite.All*

These examples should hopefully get you familiar with the Graph Powershell SDK. There is a huge number of cmdlets to explore and resources to work with.

---

And that concludes this blogpost series on getting started with Microsoft Graph. I hope you enjoyed it, thanks for reading! 

Be sure to provide any feedback on [Twitter](https://twitter.com/stianstrysse/status/1444987076085178368) or [LinkedIn](https://www.linkedin.com/posts/stianstrysse_getting-started-with-microsoft-graph-activity-6850752225954799617-c6NM).