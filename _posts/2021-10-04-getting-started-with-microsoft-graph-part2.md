---
layout: post
title: Getting started with Microsoft Graph - part 2
subtitle: The second blogpost in the series explains the Microsoft Graph.
categories: AZUREAD OFFICE365 GRAPH POWERSHELL
share-img: /assets/img/posts/2021-10-04/microsoft-graph-thumb.png
author: Stian A. Strysse
date: 2021-10-04 08:02:00
---

Let's continue with this blogpost series by looking at Microsoft Graph!

*[Part 1](/blog/getting-started-with-microsoft-graph/)* --- **Part 2** --- *[Part 3](/blog/getting-started-with-microsoft-graph-part3/)* --- *[Part 4](/blog/getting-started-with-microsoft-graph-part4/)*

+ [What is Microsoft Graph?](#what-is-microsoft-graph)
    + [API versions](#api-versions)
    + [Authentication](#authentication)
    + [Query parameters](#query-parameters)
    + [Request headers and body](#request-headers-and-body)
    + [Permission scopes and consent](#permission-scopes-and-consent)

_

## What is Microsoft Graph?

Microsoft Graph is a RESTful web API which can be used to interact with a large number of resource types in Microsoft's cloud services like Azure AD and Microsoft 365, all through a single endpoint at `https://graph.microsoft.com`:

- **Microsoft 365 core services**: Bookings, Calendar, Delve, Excel, Microsoft 365 compliance eDiscovery, Microsoft Search, OneDrive, OneNote, Outlook/Exchange, People (Outlook contacts), Planner, SharePoint, Teams, To Do, Workplace Analytics
- **Enterprise Mobility and Security services**: Advanced Threat Analytics, Advanced Threat Protection, Azure Active Directory and Intune
- **Windows 10 services**: activities, devices, notifications, Universal Print
- **Dynamics 365 Business Central**

Let's dive into the specifics you need to know to work with Microsoft Graph.

### API versions

Microsoft Graph has two API endpoint versions that you need to be aware of:

- **v1.0**: This is the officially Microsoft-supported API version and is considered as *stable*. This API version is located at `https://graph.microsoft.com/v1.0/`
- **Beta**: This is the API version where Microsoft introduces new features and breaking changes, and is not supported by Microsoft. When a new feature is introduced to Microsoft Graph, especially Public Preview features, it is usually always added to the beta version first. Beta always contains a lot of goodies that is not present in v1.0 yet. This API version is located at `https://graph.microsoft.com/beta/`

When using Microsoft Graph in automation tasks you should try to always stick with `v1.0` as this is the only Microsoft-supported API version.

### Authentication

To make a request to Microsoft Graph you are required to have an access token that you have received from the Microsoft Identity Platform, which is issued to you when you authenticate with an authorized app like e.g. Graph Explorer or Graph Powershell SDK. Access tokens are usually valid for 1 hour.

The access token contains claims information about the authenticated user, app and any permission scopes it has for the resources and APIs available through Microsoft Graph, and is used to send authorized requests to Microsoft Graph. Access tokens are encoded as JWTs (JSON Web Tokens).

You can use Graph Explorer, which is explained in the next part of this blogpost series, to extract your access token by clicking the *Access token* button below *Resource URI*.

{: .box-warning}
**Warning**: Access tokens are highly sensitive credentials and must be kept safe - only send them to trusted APIs using HTTPS.

Here's an example of an access token:

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dCI6Imk2bEdrM0ZaenhSY1ViMkMzbkVRN3N5SEpsWSIsImtpZCI6Imk2bEdrM0ZaenhSY1ViMkMzbkVRN3N5SEpsWSJ9.eyJhdWQiOiJlZjFkYTlkNC1mZjc3LTRjM2UtYTAwNS04NDBjM2Y4MzA3NDUiLCJpc3MiOiJodHRwczovL3N0cy53aW5kb3dzLm5ldC9mYTE1ZDY5Mi1lOWM3LTQ0NjAtYTc0My0yOWYyOTUyMjIyOS8iLCJpYXQiOjE1MzcyMzMxMDYsIm5iZiI6MTUzNzIzMzEwNiwiZXhwIjoxNTM3MjM3MDA2LCJhY3IiOiIxIiwiYWlvIjoiQVhRQWkvOElBQUFBRm0rRS9RVEcrZ0ZuVnhMaldkdzhLKzYxQUdyU091TU1GNmViYU1qN1hPM0libUQzZkdtck95RCtOdlp5R24yVmFUL2tES1h3NE1JaHJnR1ZxNkJuOHdMWG9UMUxrSVorRnpRVmtKUFBMUU9WNEtjWHFTbENWUERTL0RpQ0RnRTIyMlRJbU12V05hRU1hVU9Uc0lHdlRRPT0iLCJhbXIiOlsid2lhIl0sImFwcGlkIjoiNzVkYmU3N2YtMTBhMy00ZTU5LTg1ZmQtOGMxMjc1NDRmMTdjIiwiYXBwaWRhY3IiOiIwIiwiZW1haWwiOiJBYmVMaUBtaWNyb3NvZnQuY29tIiwiZmFtaWx5X25hbWUiOiJMaW5jb2xuIiwiZ2l2ZW5fbmFtZSI6IkFiZSAoTVNGVCkiLCJpZHAiOiJodHRwczovL3N0cy53aW5kb3dzLm5ldC83MmY5ODhiZi04NmYxLTQxYWYtOTFhYi0yZDdjZDAxMjIyNDcvIiwiaXBhZGRyIjoiMjIyLjIyMi4yMjIuMjIiLCJuYW1lIjoiYWJlbGkiLCJvaWQiOiIwMjIyM2I2Yi1hYTFkLTQyZDQtOWVjMC0xYjJiYjkxOTQ0MzgiLCJyaCI6IkkiLCJzY3AiOiJ1c2VyX2ltcGVyc29uYXRpb24iLCJzdWIiOiJsM19yb0lTUVUyMjJiVUxTOXlpMmswWHBxcE9pTXo1SDNaQUNvMUdlWEEiLCJ0aWQiOiJmYTE1ZDY5Mi1lOWM3LTQ0NjAtYTc0My0yOWYyOTU2ZmQ0MjkiLCJ1bmlxdWVfbmFtZSI6ImFiZWxpQG1pY3Jvc29mdC5jb20iLCJ1dGkiOiJGVnNHeFlYSTMwLVR1aWt1dVVvRkFBIiwidmVyIjoiMS4wIn0.D3H6pMUtQnoJAGq6AHd
```

Go ahead and have a look at the access token in [jwt.ms](https://jwt.ms/#access_token=eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dCI6Imk2bEdrM0ZaenhSY1ViMkMzbkVRN3N5SEpsWSIsImtpZCI6Imk2bEdrM0ZaenhSY1ViMkMzbkVRN3N5SEpsWSJ9.eyJhdWQiOiJlZjFkYTlkNC1mZjc3LTRjM2UtYTAwNS04NDBjM2Y4MzA3NDUiLCJpc3MiOiJodHRwczovL3N0cy53aW5kb3dzLm5ldC9mYTE1ZDY5Mi1lOWM3LTQ0NjAtYTc0My0yOWYyOTUyMjIyOS8iLCJpYXQiOjE1MzcyMzMxMDYsIm5iZiI6MTUzNzIzMzEwNiwiZXhwIjoxNTM3MjM3MDA2LCJhY3IiOiIxIiwiYWlvIjoiQVhRQWkvOElBQUFBRm0rRS9RVEcrZ0ZuVnhMaldkdzhLKzYxQUdyU091TU1GNmViYU1qN1hPM0libUQzZkdtck95RCtOdlp5R24yVmFUL2tES1h3NE1JaHJnR1ZxNkJuOHdMWG9UMUxrSVorRnpRVmtKUFBMUU9WNEtjWHFTbENWUERTL0RpQ0RnRTIyMlRJbU12V05hRU1hVU9Uc0lHdlRRPT0iLCJhbXIiOlsid2lhIl0sImFwcGlkIjoiNzVkYmU3N2YtMTBhMy00ZTU5LTg1ZmQtOGMxMjc1NDRmMTdjIiwiYXBwaWRhY3IiOiIwIiwiZW1haWwiOiJBYmVMaUBtaWNyb3NvZnQuY29tIiwiZmFtaWx5X25hbWUiOiJMaW5jb2xuIiwiZ2l2ZW5fbmFtZSI6IkFiZSAoTVNGVCkiLCJpZHAiOiJodHRwczovL3N0cy53aW5kb3dzLm5ldC83MmY5ODhiZi04NmYxLTQxYWYtOTFhYi0yZDdjZDAxMjIyNDcvIiwiaXBhZGRyIjoiMjIyLjIyMi4yMjIuMjIiLCJuYW1lIjoiYWJlbGkiLCJvaWQiOiIwMjIyM2I2Yi1hYTFkLTQyZDQtOWVjMC0xYjJiYjkxOTQ0MzgiLCJyaCI6IkkiLCJzY3AiOiJ1c2VyX2ltcGVyc29uYXRpb24iLCJzdWIiOiJsM19yb0lTUVUyMjJiVUxTOXlpMmswWHBxcE9pTXo1SDNaQUNvMUdlWEEiLCJ0aWQiOiJmYTE1ZDY5Mi1lOWM3LTQ0NjAtYTc0My0yOWYyOTU2ZmQ0MjkiLCJ1bmlxdWVfbmFtZSI6ImFiZWxpQG1pY3Jvc29mdC5jb20iLCJ1dGkiOiJGVnNHeFlYSTMwLVR1aWt1dVVvRkFBIiwidmVyIjoiMS4wIn0.D3H6pMUtQnoJAGq6AHd) which is a Microsoft-hosted site for checking access tokens locally. The page will translate the encoded token into human-readable values and describe them on the *Claims* tab.

### Query parameters

Optional query parameters can be used with Microsoft Graph to customize Graph responses. The support for query parameters varies from one API operation to another, and depending on the API, can differ between the `v1.0` and `beta` API endpoint versions. They only work on `GET` operations.

The query parameters we will be using in the examples in this blogpost series is:

| Query Parameter | Description                                             | Example                                                                         |
|:----------------|---------------------------------------------------------|--------------------------------------------------------------------------------:|
| $count          | Retrieves the total count of matching resources         | https://graph.microsoft.com/v1.0/users?$count=true                              |
| $expand         | Retrieves related resources (reference attributes)      | https://graph.microsoft.com/v1.0/groups?$expand=members                         |
| $filter         | Filters results (rows)                                  | https://graph.microsoft.com/v1.0/users?$filter=accountEnabled+eq+false          |
| $select         | Filters properties (columns)                            | https://graph.microsoft.com/v1.0/users?$select=id,displayName,userPrincipalName |
| $top            | Specifies the max number of objects to return per page  | https://graph.microsoft.com/v1.0/groups?$top=5                                  |

You can add several query parameters to one request if necessary, join with `&`:

```
GET https://graph.microsoft.com/v1.0/users?$filter=accountEnabled eq false&$count=true&$expand=manager
```

Using operators like `ge` (greater than), `le` (less than) and certain other specific operators is considered [advanced Graph queries](https://docs.microsoft.com/en-us/graph/aad-advanced-queries). Such queries require **Request header** set to `ConsistencyLevel: eventual`, and `$count=true` as a query parameter.

There are additional query parameters available, like `$format`, `$orderby`, `$search` and `$skip`. The query parameters are compatible with the [OData V4 query language](https://docs.oasis-open.org/odata/odata/v4.0/errata03/os/complete/part2-url-conventions/odata-v4.0-errata03-os-part2-url-conventions-complete.html#_Toc453752356).

### Request headers and body

When invoking requests to Microsoft Graph you are required to specify **Request headers**. The headers must contain at least the access token for authorization to work, or else the request will be rejected. 

When invoking requests like `POST`, `PATCH` and `PUT` you will usually be required to specify the **Request body**, this is the request data that e.g. tells Microsoft Graph to update the JobTitle for a user to a new value. When doing such operations you will also be required to add `Content-type: application/json` in the **Request headers**, so that Microsoft Graph will understand the data sent in the request body.

```
PATCH https://graph.microsoft.com/v1.0/users/0ec56567-f46b-4a06-aec1-3281eae7f158
Authorization = "Bearer <access_token>"
Content-type: application/json

{
    "jobTitle": "New title"
}
```

Using tools like Graph Explorer and Graph Powershell SDK helps you out with the authorization part, and adds the authorization header automatically. But if you make a request to Microsoft Graph directly in e.g. Powershell using `Invoke-WebRequest` or `Invoke-RestMethod` you have to include the **Request headers** or else the request will fail.

### Permission scopes and consent

Microsoft Graph integrates with the Microsoft identity platform, and the platform follows an authorization model based on [OAuth 2.0 authorization protocol](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-v2-protocols) which gives users and admins control over how data can be accessed.

Applications integrating with the Microsoft identity platform rely on [consent](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-permissions-and-consent#consent-types) in order to gain access to necessary resources or APIs, like individual user consent or tenant-wide admin consent. This means that a user can be asked to self-consent to an app's permission requirements granting the app read-access to the consenting user's profile only, or an admin can be required to do tenant-wide consent on behalf of all their users. 

Microsoft Graph exposes granular permissions, called scopes, which controls the access that apps have to resources like user accounts, groups, devices, mailboxes etc. This means that an app, like Graph Explorer or Graph Powershell SDK, can be granted specific permission scopes to only receive the required privileges (least-privilegied). However, quite a few of the permission scopes are wide - meaning that a permission scope like `User.ReadWrite.All` will grant the app read/write-access to **all** accounts in Azure AD. 

There are two types of permission scopes:

1. **Delegated permissions**: requires a signed-in user present. The app is delegated with the permissions to act as a signed-in user when it makes calls to the target resource (e.g. modify an Azure AD user). The app's effective permissions for this permission scope type are the least-privilegied intersection of the delegated permissions the app has been granted, and the privileges of the currently signed-in user. The app can never have more privilegies than the signed-in user (if there are no additional application permissions present).

    Some delegated permissions can be consented to by users, but some high-privileged delegated permissions require [admin consent](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-permissions-and-consent#admin-restricted-permissions).
2. **Application permissions**: does not require a signed-in user, and is usually granted to automation tasks and services. The app's effective permissions for this permission scope type are the full level of privileges granted by the scope permission. 

    All application permissions require [admin consent](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-permissions-and-consent#admin-restricted-permissions).

    {: .box-warning}
    **Warning**: Be particularly careful when granting application permission scopes. Any service or person with the knowledge of a key or secret in an app registration which has been granted such a permission scope, can effectively escalate their privilegies due to this type of permission, as the scope does not require the privilegies of a signed-in user. 

A permission scope like `User.ReadWrite.All` means:

- Affected resource: User
- Type of privilege: Read and write
- Scope: All users
- Effective permission: Read and write all users

A permission scope like `User.Read` means:

- Affected resource: User
- Type of privilege: Read
- Scope: The currently signed-in user
- Effective permission: Authenticate and read the currently signed-in user

{: .box-note}
**Note**: Refer to the excellent [Microsoft Graph Permission Explorer](https://graphpermissions.merill.net/index.html) for an overview of all permission scopes available for Microsoft Graph, complete with descriptions. Also, [Microsoft Docs for Microsoft Graph](https://docs.microsoft.com/en-us/graph/) contains a lot of valuable information, examples and required permission scopes for the various graph requests.

Now let's work with what we've learned so far in Microsoft's Graph Explorer.

---

Go to **[Part 3: What is Graph Explorer?](/blog/getting-started-with-microsoft-graph-part3/)**