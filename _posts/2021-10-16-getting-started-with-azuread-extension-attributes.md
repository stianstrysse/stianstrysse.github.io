---
layout: post
title: Getting started with Azure AD extension attributes
subtitle: Learn how to add custom extension attributes to Azure AD objects
categories: AZUREAD GRAPH POWERSHELL
share-img: /assets/img/posts/2021-10-16/aad-ext-attr-thumb.png
thumbnail-img: /assets/img/posts/2021-10-16/aad-ext-attr-thumb.png
author: Stian A. Strysse
---

If you need to populate values on Azure AD objects like users and groups, but there are no available attributes in the default Azure AD schema fit for the purpose, an easy solution is to add custom extension attributes to an Application object (app registration) and then populate the attributes with values on objects in Azure AD.

An example scenario is that you need to store some form of object lifecycle state value on an Azure AD object, like `Active`, `Inactive` or `PendingDeletion`, to use in reports and identity automation tasks.

The custom extension attributes can be used with the following Azure AD object types: User, Group Organization, Device and Application.

{: .box-warning}
**Warning**: Never store sensitive information in attributes in Azure AD, as all users and applications can access the values.

## Create a new app registration

It's a good choice to create a new app registration for the purpose of implementing custom extension attributes.

1. Go to the [Azure AD Portal](https://aad.portal.azure.com), click *Azure Active Directory* and *App registrations*.
2. Click *New registration*, give the app a name like **IAM Custom Extension Attributes**, keep the other settings default and click *Register*.
3. Make a note of the app registration's `Object ID` as we need this value when creating the extension attributes.

We'll use Microsoft Graph via Graph Explorer to add the custom extension attributes to the app registration, but you can of course use [Aure AD Powershell](https://docs.microsoft.com/en-us/powershell/azure/active-directory/using-extension-attributes-sample?view=azureadps-2.0) or Microsoft Graph Powershell SDK too. If you need to learn how to work with Microsoft Graph and Graph Explorer, check out my blogpost series [Getting started with Microsoft Graph](https://learningbydoing.cloud/blog/getting-started-with-microsoft-graph/).

## Add custom extension attribute in Graph Explorer

Custom extension attributes can be of the following types: `Binary`, `Boolean` (true/false), `DateTime` (2021-10-16T18:01:29), `String` ("Some Value"), `Integer` (12345) and `LargeInteger`.

1. Go to [Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer).
2. Do a `GET` request to resource Uri `https://graph.microsoft.com/v1.0/applications/{App registration Object ID}` - replace `{App registration Object ID}` with the actual objectId of the app registration created earlier, and click *Run query*. You should get the app registration returned in the response.

    ```
    GET https://graph.microsoft.com/v1.0/applications/d74460c9-27d2-4dc9-bcba-49087063c360
    ```
3. Now change method to `POST` with resource Uri `https://graph.microsoft.com/v1.0/applications/{App registration Object ID}` and add `Request body` as shown below (in this example we create a `String` attribute named `ObjectLifeCycleState` for User and Group objects):

    ```
    POST https://graph.microsoft.com/v1.0/applications/d74460c9-27d2-4dc9-bcba-49087063c360/extensionProperties

    {
        "name": "ObjectLifeCycleState",
        "dataType": "string",
        "targetObjects": [
            "User",
            "Group"
        ]
    }
    ```

![Graph Explorer Create Ext Attribute](/assets/img/posts/2021-10-16/ms-graph-create-extattr.png)

Make a note of the `name` value in the response, as this is the full attribute name that can now be populated with a string value on User and Group objects in Azure AD. The attribute name consists of `extension_` + `Application (client) ID` + `attribute name`, which in this example is `extension_cde0e9a5d3f44a81b81097334dbb9f66_ObjectLifeCycleState`.

## Populate the custom extension attribute on a User object

Now that we have created the custom extension attribute on an application and it is available for User and Group objects in Azure AD, we can go ahead and populate the attribute with a value.

- In [Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer), set method to `Patch` with resource Uri `https://graph.microsoft.com/v1.0/users/{user objectId or upn}}`, insert the objectId or UserPrincipalName of an existing user. Add `Request body` and set the extension attribute name to a value.

    ```
    PATCH https://graph.microsoft.com/v1.0/users/e600712c-2132-455f-8d9f-ae0fc5ac9abe

    {
        "extension_cde0e9a5d3f44a81b81097334dbb9f66_ObjectLifeCycleState": "Active"
    }
    ```

![Graph Explorer Patch Ext Attribute](/assets/img/posts/2021-10-16/ms-graph-patch-extattr.png)

If you get a HTTP 204 response (no content), the patch was successfull.

## Retrieve custom extension attribute on a User object

You can now retrieve the custom extension attribute on the User object.

- In [Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer), set method to `Get` with resource Uri `https://graph.microsoft.com/v1.0/users/{user objectId or upn}`, insert the objectId or UserPrincipalName of the user you populated the extension attribute for, add `id`, `displayName`, `userPrincipalName` and the custom extension attribute name to the `select` query parameter.

    ```
    GET https://graph.microsoft.com/v1.0/users/e600712c-2132-455f-8d9f-ae0fc5ac9abe?$select=id,displayName,userprincipalname,extension_cde0e9a5d3f44a81b81097334dbb9f66_ObjectLifeCycleState
    ```

![Graph Explorer Get Ext Attribute](/assets/img/posts/2021-10-16/ms-graph-get-extattr.png)

And that concludes this blog post, thanks for reading!

Be sure to provide any feedback on [Twitter](https://twitter.com/stianstrysse/status/1449486596957409282) or [LinkedIn](https://www.linkedin.com/posts/stianstrysse_getting-started-with-azure-ad-extension-attributes-activity-6855253263042801665-6UV5).