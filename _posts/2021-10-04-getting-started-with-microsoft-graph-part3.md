---
layout: post
title: Getting started with Microsoft Graph - part 3
subtitle: The third blogpost in the series explains the Graph Explorer.
categories: AZUREAD OFFICE365 GRAPH POWERSHELL
share-img: /assets/img/posts/2021-10-04/microsoft-graph-thumb.png
author: Stian A. Strysse
date: 2021-10-04 08:01:00
---

Let's continue with this blogpost series by looking at Graph Explorer!

*[Part 1](/blog/getting-started-with-microsoft-graph/)* --- *[Part 2](/blog/getting-started-with-microsoft-graph-part2/)* --- **Part 3** --- *[Part 4](/blog/getting-started-with-microsoft-graph-part4/)*

+ [What is Graph Explorer?](#what-is-graph-explorer)
    + [Azure AD testing environment](#azure-ad-testing-environment)
    + [Consenting to permission scopes in Graph Explorer](#consenting-to-permission-scopes-in-graph-explorer)
    + [Working with Graph Explorer](#working-with-graph-explorer)
    + [Request examples for user objects](#request-examples-for-user-objects)
    + [Request examples for group objects](#request-examples-for-group-objects)

_

## What is Graph Explorer?

[Graph Explorer](https://aka.ms/ge) is a tool hosted by Microsoft which you can use to explore and work with Microsoft Graph in a web portal. When you first visit the website, the Graph Explorer tool is connected to Microsoft's demo tenant that you can use to try out varius REST API calls to interact with resources like Users, Groups, Devices, and a lot more. 

The cool thing is that you can also log in to Graph Explorer with an Azure AD account to be able to use the tool with Microsoft Graph in your own Azure AD tenant. Graph Explorer is an excellent tool for getting started with Microsoft Graph!

{: .box-note}
**Note:** As always when learning new things, do NOT use a production environment to test out stuff if you don't have full control of what you're doing. Instead, either use the provided demo environment in Graph Explorer, or have a look at the options in the chapter below!

![Graph Explorer](/assets/img/posts/2021-10-04/graph-explorer.png)

You can use Graph Explorer to extract your access token by clicking the *Access token* button below *Resource URI*.

{: .box-warning}
**Warning**: Access tokens are highly sensitive credentials and must be kept safe - only send them to trusted APIs using HTTPS.

### Azure AD testing environment

If you don't already have an Azure AD testing environment you can play around with, you have a few options to get this sorted out.

- **Option 1**: Sign up for an [Azure subscription and Azure AD Premium trial](https://azure.microsoft.com/en-us/trial/get-started-active-directory/). Note that this option does not provide any Office or Microsoft 365 licenses.
- **Option 2**: Sign up with the [Microsoft 365 Developer Program](https://developer.microsoft.com/en-us/microsoft-365/dev-program). This option grants a 25-seat Microsoft 365 E5 developer subscription valid for 90 days (renewable), which means you can play around with all the goodies offered by Microsoft's E5 SKU. You can also choose to populate the tenant with demo users and content. This option does not provide an Azure subscription.
- **Option 3**: If you work in a company registered with the Microsoft Partner Network (MPN), you can create demo environments at [http://demos.microsoft.com](http://demos.microsoft.com). This option does not provide an Azure subscription.

### Consenting to permission scopes in Graph Explorer

The first time you sign in to Graph Explorer with an Azure AD admin account in a testing environment, you will have to consent to Graph permissions scopes to get things to work. Many scopes will require an admin user's consent due to the privilegies the scopes grant (e.g. Users.Read.All which grants permission to read all users in the tenant).

{: .box-note}
**Note:** Any permission scopes you consent to for Graph Explorer is only [delegated permissions type](/blog/getting-started-with-microsoft-graph-part2/#permission-scopes-and-consent), which means that Graph Explorer will never have more privilegies than the siged-in user, and the signed-in user must be present. 

1. Go to Graph Exporer at [https://aka.ms/ge](https://aka.ms/ge), click **Sign in to Graph Explorer** under **Authentication** and authenticate with an admin account in your test tenant.
2. At the consent screen you will see that Graph Explorer requests permission scopes to sign you in and read your profile. click **Accept**. Note that you can check the **Consent on behalf of your organization** if you want to, which basically means that another user in the same tenant will not have to do the consent again when signing in to Graph Explorer for the first time. This step will create Graph Explorer's [Service Principal](https://docs.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals#service-principal-object) in your tenant if not already present (see Azure AD portal -> Enterprise Applications -> Graph Explorer (official site)).
    
    ![Graph Explorer consent 1](/assets/img/posts/2021-10-04/graph-explorer-consent1.png)

3. As you are now signed in and have granted Graph Explorer permissions to read your profile, you can do a [GET request](/blog/getting-started-with-microsoft-graph/#what-is-a-rest-api) to your own profile with the pre-populated resource URI `https://graph.microsoft.com/v1.0/me` by clicking **Run query**. A response with your profile in JSON format will appear in the **Response preview** tab:

    ```json
    {
        "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users/$entity",
        "@odata.id": "https://graph.microsoft.com/v2/eeb7df0a-3cca-4908-bd1d-3a11a20118ae/directoryObjects/6c834590-0d25-4863-8c05-a6dde9be0812/Microsoft.DirectoryServices.User",
        "businessPhones": [
            "425-555-0100"
        ],
        "displayName": "MOD Administrator",
        "givenName": "MOD",
        "jobTitle": null,
        "mail": "admin@M365x955081.OnMicrosoft.com",
        "mobilePhone": "425-555-0101",
        "officeLocation": null,
        "preferredLanguage": "en-US",
        "surname": "Administrator",
        "userPrincipalName": "admin@M365x955081.onmicrosoft.com",
        "id": "6c834590-0d25-4863-8c05-a6dde9be0812"
    }
    ```

    If you used Graph API version **v1.0** you can now try to change to **beta** in the dropdown, click **Run query** and see if you spot any differences. Depending on the type of resource returned (in this case an Azure AD user), v1.0 will return certain attributes, while beta will return all attributes, including those with `null` values.

4. Let's proceed with trying to GET **all users** in the tenant by changing the resource URI to `https://graph.microsoft.com/v1.0/users`, click **Run query**. Since Graph Explorer currently has no delegated permission scopes to read other user accounts than your own, [HTTP status code](/blog/getting-started-with-microsoft-graph/#what-is-a-http-status-code) **403 Forbidden** is returned by the API. 

    ![Graph Explorer 403](/assets/img/posts/2021-10-04/graph-explorer-403forbidden.png)

    To consent to the required permission scope, go to the **Modify permissions** tab below the resource URI. Graph Explorer will try to list the required permissions for the current resource URI as best as it can. In this specific scenario, where you try to read all users in the tenant, the required permission scope is **User.Read.All** - so go ahead and locate it in the list and click **Consent**. 
    
    Once again you will be presented with the admin consent dialogue, but this time you will see that Graph Explorer asks for permission to **Read all users' full profiles** (User.Read.All). Click **Accept**.

    ![Graph Explorer consent 2](/assets/img/posts/2021-10-04/graph-explorer-consent2.png)

5. When you have consented to the new permission scope, click **Run query** again. This time you should see all users in the tenant returned in the response, at least the first 100 objects. 

    ```json
    {
    "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users",
    "value": [
        {
            "@odata.id": "https://graph.microsoft.com/v2/eeb7df0a-3cca-4908-bd1d-3a11a20118ae/directoryObjects/299f9843-a399-432b-bd03-bc7980e7a20a/Microsoft.DirectoryServices.User",
            "businessPhones": [],
            "displayName": "Conf Room Adams",
            "givenName": null,
            "jobTitle": null,
            "mail": "Adams@M365x955081.OnMicrosoft.com",
            "mobilePhone": null,
            "officeLocation": null,
            "preferredLanguage": null,
            "surname": null,
            "userPrincipalName": "Adams@M365x955081.OnMicrosoft.com",
            "id": "299f9843-a399-432b-bd03-bc7980e7a20a"
        },
        {
            "@odata.id": "https://graph.microsoft.com/v2/eeb7df0a-3cca-4908-bd1d-3a11a20118ae/directoryObjects/0cf215e7-b99f-4013-bd49-4a68060efed2/Microsoft.DirectoryServices.User",
            "businessPhones": [
                "+1 425 555 0109"
            ],
            "displayName": "Adele Vance",
            "givenName": "Adele",
            "jobTitle": "Retail Manager",
            "mail": "AdeleV@M365x955081.OnMicrosoft.com",
            "mobilePhone": null,
            "officeLocation": "18/2111",
            "preferredLanguage": null,
            "surname": "Vance",
            "userPrincipalName": "AdeleV@M365x955081.OnMicrosoft.com",
            "id": "0cf215e7-b99f-4013-bd49-4a68060efed2"
        },
        ...
    ```

Go to *[Azure AD portal](https://aad.portal.azure.com) -> Enterprise Applications -> Graph Explorer (official site)* and click the **Permissions** tab to see all the consented permission scopes for single users (user consent) or all users (admin consent).

![Graph Explorer consented permissions](/assets/img/posts/2021-10-04/graph-explorer-consented-permissions.png)

Note that you can click the three dots to the right of your signed-in user in the Graph Explorer menu and click **Select permissions** to get an overview of all permission scopes available in Graph. Use the [Microsoft Graph Permission Explorer](https://graphpermissions.merill.net/index.html) website to identify permissions. Also refer to Microsoft's official [Graph documentation](https://docs.microsoft.com/en-us/graph/) which will describe necessary permission scopes (from least to most privileged) and show examples on how to complete various Graph requests.

Now you know how to consent to delegated permission scopes in Graph Explorer. Let's continue with some examples on what you can do with Graph Explorer!

### Working with Graph Explorer

I'll showcase various requests that can be executed using Graph Explorer - complete with HTTP method, resource URI, request headers and body (where applicable), query parameters and required permission scopes.

The examples are set up like this (similar to how Microsoft Docs show Graph request examples):

```
PATCH https://graph.microsoft.com/v1.0/me
Content-type: application/json

{
  "businessPhones": [
    "+1 425 555 0109"
  ],
  "officeLocation": "My Site"
}
```

What it means:

![Graph Explorer example 1](/assets/img/posts/2021-10-04/graph-explorer-example1.png)

Add it to Graph Explorer:

1. Set correct **HTTP method**
2. Paste in **Resource URI** and set correct **API version**
3. Paste in **Request body** (if applicable)
4. Add **Request headers** (if applicable)

    {: .box-note}
    **Note**: Adding `Content-type: application/json` in **Request headers** in Graph Explorer is not necessary as it will be applied automatically. But other headers listed in these examples must be added (e.g. `ConsistencyLevel: eventual`)


![Graph Explorer example 1](/assets/img/posts/2021-10-04/graph-explorer-example2.png)

Let's dive into some examples to see how it all works.

{: .box-warning}
**Warning**: As some of these operations will do actual changes in your tenant, be sure to use an [Azure AD testing environment](/blog/getting-started-with-microsoft-graph-part3/#azure-ad-testing-environment) when trying out stuff!

### Request examples for user objects

- **Fetch a specific user object**

    Make a GET request to `/users/{upn | id}` endpoint, objects like users can be referenced directly by specifying 'userPrincipalName' or 'objectId' in the **Resource URI**.

    ```
    GET https://graph.microsoft.com/v1.0/users/admin@M365x955081.onmicrosoft.com
    ```

    ```
    GET https://graph.microsoft.com/v1.0/users/6c834590-0d25-4863-8c05-a6dde9be0812
    ```

    *Permission scopes required: User.Read.All*

- **Fetch selected attributes on a user object**

    Make a GET request to `/users` endpoint and utilize the `$select` query parameter to specify which attributes to return in the response.

    ```
    GET https://graph.microsoft.com/v1.0/users/admin@M365x955081.onmicrosoft.com?$select=id,userPrincipalName,displayName,jobTitle,manager
    ```

    *Permission scopes required: User.Read.All*

- **Fetch and expand reference attributes on a user object**

    Make a GET request to `/users` endpoint and utilize the `$expand` query parameter to specify a reference attribute to expand, like 'manager'. If you only want to return specific attributes on the manager object, add them to a `$select` query parameter.

    ```
    GET https://graph.microsoft.com/v1.0/users/admin@M365x955081.onmicrosoft.com?$expand=manager($select=id,userPrincipalName,displayName,jobTitle)
    ```

    *Permission scopes required: User.Read.All*

- **Fetch the first x number of users**

    Make a GET request to `/users` endpoint and utilize the `$top` query parameter to specify the maximum number of objects to return. Note that 999 is max, and the default is 100 if `$top` is not specified.

    ```
    GET https://graph.microsoft.com/v1.0/users?$top=5
    ```

    *Permission scopes required: User.Read.All*

- **Fetch user(s) with a specific attribute value**

    Make a GET request to `/users` endpoint and utilize the `$filter` query parameter to filter the response to only return objects with a specific attribute value.

    ```
    GET https://graph.microsoft.com/v1.0/users?$filter=displayName eq 'MOD Admin'
    ```

     ```
    GET https://graph.microsoft.com/v1.0/users?$filter=accountEnabled eq true
    ```

    *Permission scopes required: User.Read.All*

- **Fetch users and count the number of returned objects**

    Make a GET request to `/users` endpoint and utilize the `$count` query parameter to retrieve a count of returned objects. Note that **Request header** `ConsistencyLevel: eventual` is required for this operation.

    ```
    GET https://graph.microsoft.com/v1.0/users?$count=true
    ConsistencyLevel: eventual
    ```

    *Permission scopes required: User.Read.All*

- **Fetch users created within a specific time period**

    Make a GET request to `/users` endpoint and utilize the `$filter` query parameter with dates to specify time periods to filter the response on. Note that **Request header** `ConsistencyLevel: eventual` and `$count=true` query parameter is required for this operation, as is true for all [advanced Graph queries](https://docs.microsoft.com/en-us/graph/aad-advanced-queries) using `ge` (greater than), `le` (less than) and certain other specific operators.

    ```
    GET https://graph.microsoft.com/v1.0/users?$filter=createdDateTime ge 2021-09-17T07:00:00Z and createdDateTime le 2021-09-18T07:00:00Z&$count=true
    ConsistencyLevel: eventual
    ```

    *Permission scopes required: User.Read.All*

- **Create a new user**

    Make a POST request to `/users` endpoint to create a new user account. The attributes in the example below are the very minimum of required attributes, add additional attributes in **Request body** as needed. To generate a strong password, utilize [this powershell script](https://github.com/stianstrysse/powershell-scripts/blob/main/StrongPasswordGenerator.ps1).

    ```
    POST https://graph.microsoft.com/v1.0/users
    Content-type: application/json

    {
        "accountEnabled": false,
        "displayName": "Graph Demo User",
        "mailNickname": "GraphDemoUser",
        "userPrincipalName": "graphdemouser@M365x955081.onmicrosoft.com",
        "passwordProfile" : {
            "forceChangePasswordNextSignIn": true,
            "password": "at8oO5SqYfLFcgm$-NPVj&H2y"
        }
    }
    ```

    *Permission scopes required: User.ReadWrite.All*

- **Update an existing user**

    Make a PATCH request to `/users/{upn | id}` endpoint to change data on an existing user account, add the correct UPN or objectId in the **Resource URI**. Only include attributes that you want to change in the in **Request body**.

    ```
    PATCH https://graph.microsoft.com/v1.0/users/graphdemouser@M365x955081.onmicrosoft.com
    Content-type: application/json

    {
        "accountEnabled": true,
        "displayName": "Graph Demo User Updated",
        "givenName": "Demo",
        "surname": "Test"
    }
    ```

    *Permission scopes required: User.ReadWrite.All*

- **Delete an existing user**

    Make a DELETE request to `/users/{upn | id}` endpoint to delete an existing user account, add the correct UPN or objectId in the **Resource URI**. The account will be soft-deleted for 30 days before Azure AD automatically deletes it permanently.

    ```
    DELETE https://graph.microsoft.com/v1.0/users/graphdemouser@M365x955081.onmicrosoft.com
    ```

    *Permission scopes required: User.ReadWrite.All*

- **Fetch deleted users**

    Make a GET request to `/directory/deletedItems/microsoft.graph.user` endpoint to fetch all soft-deleted users in the tenant.

    ```
    GET https://graph.microsoft.com/v1.0/directory/deletedItems/microsoft.graph.user
    ```

    *Permission scopes required: User.Read.All*

- **Restore a deleted user**

    Make a POST request to `/directory/deletedItems/{id}/restore` endpoint to restore a soft-deleted user, add the correct objectId in the **Resource URI**.

    ```
    POST https://graph.microsoft.com/v1.0/directory/deletedItems/86046a93-f6dd-4482-afa4-6eb0ed669123/restore
    Content-type: application/json
    ```

    *Permission scopes required: User.ReadWrite.All*

### Request examples for group objects

- **Fetch a specific group object**

    Make a GET request to `/groups/{id}` endpoint, objects like groups can be referenced directly by specifying the 'objectId' in the **Resource URI**.

    ```
    GET https://graph.microsoft.com/v1.0/groups/2025d2e8-4c4b-4c30-9488-48e51e21e8d1
    ```

    *Permission scopes required: Group.Read.All*

- **Fetch and expand members on a group object**

    Make a GET request to `/groups/{id}` endpoint and utilize the `$expand` query parameter to include data on group members in the response.

    ```
    GET https://graph.microsoft.com/v1.0/groups/7991df10-5597-4d63-9040-b66659b6629d?$expand=members
    ```

    *Permission scopes required: Group.Read.All*

- **Add a user as a group member**

    Make a POST request to `/groups/{group-id}/members/$ref` endpoint, add the objectId of the group in the **Resource URI** and add the objectId of the user in the **Request body** to add the user as a member in the specified group.

    ```
    POST https://graph.microsoft.com/v1.0/groups/7991df10-5597-4d63-9040-b66659b6629d/members/$ref
    Content-type: application/json

    {
        "@odata.id": "https://graph.microsoft.com/v1.0/directoryObjects/0cf215e7-b99f-4013-bd49-4a68060efed2"
    }
    ```

    *Permission scopes required: GroupMember.ReadWrite.All or Group.ReadWrite.All*

- **Remove a user as a group member**

    Make a DELETE request to `/groups/{group-id}/members/{user-id}/$ref` endpoint, add the objectId of the group and user in the **Resource URI** to remove the user as a member in the specified group.

    ```
    DELETE https://graph.microsoft.com/v1.0/groups/7991df10-5597-4d63-9040-b66659b6629d/members/0cf215e7-b99f-4013-bd49-4a68060efed2/$ref
    ```

    *Permission scopes required: GroupMember.ReadWrite.All or Group.ReadWrite.All*

- **Add a user as a group owner**

    Make a POST request to `/groups/{group-id}/owners/$ref` endpoint, add the objectId of the group in the **Resource URI** and add the objectId of the user in the **Request body** to add the user as an owner in the specified group.

    ```
    POST https://graph.microsoft.com/v1.0/groups/7991df10-5597-4d63-9040-b66659b6629d/owners/$ref
    Content-type: application/json

    {
        "@odata.id": "https://graph.microsoft.com/v1.0/directoryObjects/0cf215e7-b99f-4013-bd49-4a68060efed2"
    }
    ```

    *Permission scopes required: Group.ReadWrite.All*

- **Remove a user as a group owner**

    Make a DELETE request to `/groups/{group-id}/owners/{user-id}/$ref` endpoint, add the objectId of tbe group and user in the **Resource URI** to remove the user as an owner of the specified group.

    ```
    DELETE https://graph.microsoft.com/v1.0/groups/7991df10-5597-4d63-9040-b66659b6629d/owners/0cf215e7-b99f-4013-bd49-4a68060efed2/$ref
    ```

    *Permission scopes required: Group.ReadWrite.All*

- **Create a new static Azure AD security group**

    Make a POST request to `/groups` endpoint to create a new security group with static membership in Azure AD, including owner. The attributes in the example below are the very minimum of required attributes (except `owners@odata.bind` which can be skipped to not set an owner), add additional attributes in **Request body** as needed.

    ```
    POST https://graph.microsoft.com/v1.0/groups
    Content-type: application/json

    {
        "displayName": "Graph Test Group 1",
        "mailNickname": "GraphTestGroup1",
        "mailEnabled": false,
        "groupTypes": [
        ],
        "securityEnabled": true,
        "owners@odata.bind": [
            "https://graph.microsoft.com/v1.0/users/0cf215e7-b99f-4013-bd49-4a68060efed2"
        ]
    }
    ```

    *Permission scopes required: Group.ReadWrite.All*

- **Create a new dynamic Azure AD security group**

    Make a POST request to `/groups` endpoint to create a new security group with dynamic membership (`groupTypes: DynamicMembership`). The attributes in the example below are the very minimum of required attributes, add additional attributes in **Request body** as needed.

    ```
    POST https://graph.microsoft.com/v1.0/groups
    Content-type: application/json

    {
        "displayName": "Graph Test Group 2 dynamic",
        "mailNickname": "GraphTestGroup2dynamic",
        "mailEnabled": false,
        "groupTypes": [
            "DynamicMembership"
        ],
        "securityEnabled": true,
        "membershipRule": "user.accountEnabled -eq true",
        "membershipRuleProcessingState": "On"
    }
    ```

    *Permission scopes required: Group.ReadWrite.All*

- **Create a new Microsoft 365 group**

    Make a POST request to `/groups` endpoint to create a new Microsoft 365 group (`groupTypes: Unified`). The attributes in the example below are the very minimum of required attributes, add additional attributes in **Request body** as needed.

    ```
    POST https://graph.microsoft.com/v1.0/groups
    Content-type: application/json

    {
        "displayName": "Graph Test M365 Group",
        "mailNickname": "GraphTestGroup3",
        "mailEnabled": true,
        "groupTypes": [
            "Unified"
        ],
        "securityEnabled": false
    }
    ```

    *Permission scopes required: Group.ReadWrite.All*

- **Teams-enable an existing Microsoft 365 group**

    Make a PUT request to `/groups/{group-id}/team` endpoint to Teams-enable an existing Microsoft 365 group, add the correct objectId in the **Resource URI**. Groups created outside of Teams is not Teams-enabled by default. If the group was just created, you may have to wait 15 minutes before you can Teams-enable it. Note that a Teams-enabled owner must be present on the group.

    ```
    POST https://graph.microsoft.com/v1.0/groups/82633cc4-c5bb-4d46-b399-8bffb9560ede/team
    Content-type: application/json

    {  
        "memberSettings": {
            "allowCreatePrivateChannels": true,
            "allowCreateUpdateChannels": true
        },
        "messagingSettings": {
            "allowUserEditMessages": true,
            "allowUserDeleteMessages": true
        },
        "funSettings": {
            "allowGiphy": true,
            "giphyContentRating": "strict"
        }
    }
    ```

    *Permission scopes required: Group.ReadWrite.All*

- **Update an existing group**

    Make a PATCH request to `/groups/{group-id}` endpoint to change data on an existing group, add the correct objectId in the **Resource URI**. Only include attributes that you want to change in the in **Request body**.

    ```
    PATCH https://graph.microsoft.com/v1.0/groups/c6c01449-8405-420d-ac67-1592738cf152
    Content-type: application/json

    {
        "displayName": "Graph Test Group 1 Updated",
        "description": "Updated description"
    }
    ```

    *Permission scopes required: Group.ReadWrite.All*

- **Delete an existing group**

    Make a DELETE request to `/groups/{group-id}` endpoint to delete an existing group, add the correct objectId in the **Resource URI**. A security group will be permanently deleted immediately, a Microsoft 365 group will be soft-deleted for 30 days until Azure AD automatically deletes it permanently.

    ```
    DELETE https://graph.microsoft.com/v1.0/groups/82633cc4-c5bb-4d46-b399-8bffb9560ede
    ```

    *Permission scopes required: Group.ReadWrite.All*

- **Fetch deleted groups**

    Make a GET request to `/directory/deletedItems/microsoft.graph.group` endpoint to fetch all soft-deleted groups in the tenant.

    ```
    GET https://graph.microsoft.com/v1.0/directory/deletedItems/microsoft.graph.group
    ```

    *Permission scopes required: Group.Read.All*

- **Restore a deleted group**

    Make a POST request to `/directory/deletedItems/{id}/restore` endpoint to restore a soft-deleted group (Microsoft 365), add the correct objectId in the **Resource URI**.

    ```
    POST https://graph.microsoft.com/v1.0/directory/deletedItems/82633cc4-c5bb-4d46-b399-8bffb9560ede/restore
    Content-type: application/json
    ```

    *Permission scopes required: Group.ReadWrite.All*

These examples should hopefully get you started with Graph Explorer. Now let's jump over to look at the new Microsoft Graph Powershell SDK.

---

Go to **[Part 4: What is Microsoft Graph PowerShell SDK?](/blog/getting-started-with-microsoft-graph-part4/)**