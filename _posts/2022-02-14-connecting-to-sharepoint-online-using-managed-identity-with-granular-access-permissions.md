---
layout: post
title: Connecting to SharePoint Online using Managed Identity with granular access permissions
subtitle: Microsoft Graph and SharePoint Online supports some granular access permissions using Sites.Selected application scope in Graph, and app access role permissions in Site collections. It even works with Managed Identities.
categories: AZUREAD AZURE IDENTITY GOVERNANCE MSI MANAGEDIDENTITY
thumbnail-img: /assets/img/posts/2022-02-14/aad-managed-identity-graph-spo-thumbnail.png
author: Stian A. Strysse
---

The `Sites.Selected` application scope was [introduced in Microsoft Graph](https://devblogs.microsoft.com/microsoft365dev/controlling-app-access-on-specific-SharePoint-site-collections/) some time ago to support granular app access permissions in SharePoint Online. With this scope one can grant application access to specific SharePoint Online site collections, instead of granting access to all site collections in the tenant, and this is very welcomed granular and least-privilege functionality in Microsoft Graph - as Graph in general is based on _"all or nothing"_ permission scopes. It's a step in the right direction, and I hope we will see more of these types of scopes in Microsoft Graph in the future.

When granting `Sites.Selected` application scope to a Service Principal, it receives no access to data in SharePoint Online without further action. One must additionally grant app access within each SharePoint Online site which the Service Principal should be allowed to access, with _Read_ or _Write_ role. This can, depending on the scenario, completely remove the necessity of granting `Sites.Read.All` or `Sites.ReadWrite.All` scopes which grants access to **all** sites in the tenant.

Recently I was looking into automating Microsoft 365 tasks using Logic Apps, and being a zero-trust and least-privilege advocate I figured out that a Managed Identity with `Sites.Selected` application scope would be the best option. I've seen several blogs explaining how to use the `Sites.Selected` scope with an App Registration and Service Principal, but none of them so far described how to make it work with Managed Identities. Also, Microsoft's own [documentation](https://docs.microsoft.com/en-us/graph/api/site-get-permission?view=graph-rest-1.0&tabs=http) on this topic could be better - which is why I decided to write this blog post.

Refer to my blog posts on [Managed Identities](https://learningbydoing.cloud/blog/stop-using-client-secrets-start-using-managed-identities/) and [Microsoft Graph](https://learningbydoing.cloud/blog/getting-started-with-microsoft-graph/) if you need more information on these topics.

So, let's dive into the technicalities.

![Managed Identity SitesSelected Graph Scope](/assets/img/posts/2022-02-14/aad-managed-identity-graph-spo-thumbnail.png)

- [The scenario](#the-scenario)
- [The prerequisites](#the-prerequisites)
- [Grant application scope in Microsoft Graph](#grant-application-scope-in-microsoft-graph)
- [Grant app access to a specific SharePoint site](#grant-app-access-to-a-specific-sharepoint-site)
- [Configure Logic App to retrieve SharePoint list items](#configure-logic-app-to-retrieve-sharepoint-list-items)

## The scenario

A Logic App will be used to retrieve items from a specific SharePoint List, with the end-goal of processing the items by creating Microsoft 365 groups, setting group ownership etc. The Logic App will be granted application scopes in Microsoft Graph to be allowed to carry out these actions. Focus is on automation and least-privilege access. This blog post will only describe the first part; how to retrieve items from a SharePoint list using a Managed Identity with `Sites.Selected` application scope.

## The prerequisites

I needed the following resources:

- A Logic App with a `System assigned` Managed Identity.
- A SharePoint Site with a SharePoint list populated with a few columns and items.
- Access to grant Microsoft Graph application scopes and SharePoint site permissions.

Once the `System assigned` Managed Identity [was enabled on the Logic App](https://learningbydoing.cloud/blog/stop-using-client-secrets-start-using-managed-identities/#enable-managed-identity-for-an-azure-resource), I noted down the `Object (principal) ID` for the Managed Identity _(guid e8800382-610d-4761-9b15-873065e53227)_ - which will be used to grant `Sites.Selected` application scope in Microsoft Graph.

![Logic App MSI Principal ID](/assets/img/posts/2022-02-14/logicapp-managedidentity-principalid.png)

Visiting the **Enterprise Application** blade in the [Azure AD Portal](https://aad.portal.azure.com), I found the recently created Managed Identity object and noted down the `Application ID` for the Managed Identity _(guid 827fc69f-2814-44d7-96bc-492f2bf21c83)_ - which will be used to grant permission within the SharePoint site.

![Logic App MSI App ID](/assets/img/posts/2022-02-14/aad-managed-identity-appid.png)

I then created a Team with the name **LBD M365 Automation**, which generated a SharePoint site, and added a SharePoint List with the name **OrderList** with necessary columns and a few items.

![M365 Order List](/assets/img/posts/2022-02-14/m365-orderlist.png)

Using [Graph Explorer](https://learningbydoing.cloud/blog/getting-started-with-microsoft-graph-part3/#what-is-graph-explorer) I verified that I was able to retrieve the items in the SharePoint List.

```text
GET https://graph.microsoft.com/v1.0/sites/<tenant>.sharepoint.com:/sites/LBDM365Automation:/lists/OrderList/items?$select=id,webUrl,fields,createdDateTime&$expand=fields($select=Title,Owner,Description,AutomationCompleted)
```

![Graph Explorer Get List Items](/assets/img/posts/2022-02-14/graph-explorer-get-listitems.png)

The prerequisites are completed, now over to grant permissions.

## Grant application scope in Microsoft Graph

Adding application scopes in the Azure AD Portal GUI doesn't seem to be possible for Managed Identities, so I used **Cloud Shell** in the Azure Portal and the following Powershell code to grant the `Sites.Selected` application scope to the Logic App's Managed Identity. `$ObjectId` is set to the guid value of `Object (principal) ID` for the Managed Identity noted down earlier.

```powershell
# Add the correct 'Object (principal) ID' for the Managed Identity
$ObjectId = "e8800382-610d-4761-9b15-873065e53227"

# Add the correct Graph scope to grant
$graphScope = "Sites.Selected"

Connect-AzureAD
$graph = Get-AzureADServicePrincipal -Filter "AppId eq '00000003-0000-0000-c000-000000000000'" 
$graphAppRole = $graph.AppRoles | ? Value -eq $graphScope
New-AzureADServiceAppRoleAssignment -Id $graphAppRole.Id -PrincipalId $ObjectId -ResourceId $graph.ObjectId -ObjectID $ObjectID
```

Running the Powershell code produced the following output in the console, indicating that the scope was successfully granted.

```text
ObjectId      ResourceDisplayName PrincipalDisplayName
--------      ------------------- --------------------
ggOA6A1hYU... Microsoft Graph     lbd-m365-automation-la
```

The Managed Identity now has the `Sites.Selected` application scope in Microsoft Graph, but still requires app access within the specific SharePoint site.

## Grant app access to a specific SharePoint site

Using [Graph Explorer](https://learningbydoing.cloud/blog/getting-started-with-microsoft-graph-part3/#what-is-graph-explorer) I added **app access** with **write** role for the Logic App's Managed Identity to the specific SharePoint Online site in Microsoft Graph. `id` in the `body` of the request is set to the guid value of `Application ID` for the Managed Identity noted down earlier.

```json
POST https://graph.microsoft.com/v1.0/sites/<tenant>.sharepoint.com:/sites/LBDM365Automation:/permissions
Content-Type: application/json

{
    "roles": [
        "write"
    ],
    "grantedToIdentities": [
        {
            "application": {
                "id": "827fc69f-2814-44d7-96bc-492f2bf21c83",
                "displayName": "lbd-m365-automation-la"
            }
        }
    ]
}
```

Graph responded with `Created - 201` status code and the following response body.

```json
{
    "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#Collection(microsoft.graph.permission)/$entity",
    "id": "aTowaS50fG1zLnNwLmV4dHw4MjdmYzY5Zi0yODE0LTQ0ZDctOTZiYy00OTJmMmJmMjFjODNANTZhYTBmYTUtYjdkZC00ZGJjLTgxZGEtM2Q1N2IyZDc5ZmZm",
    "roles": [
        "write"
    ],
    "grantedToIdentitiesV2": [
        {
            "application": {
                "displayName": "lbd-m365-automation-la",
                "id": "827fc69f-2814-44d7-96bc-492f2bf21c83"
            }
        }
    ],
    "grantedToIdentities": [
        {
            "application": {
                "displayName": "lbd-m365-automation-la",
                "id": "827fc69f-2814-44d7-96bc-492f2bf21c83"
            }
        }
    ]
}
```

The Logic App's Managed Identity should now have enough permissions to both read and write the SharePoint List items via Microsoft Graph.

## Configure Logic App to retrieve SharePoint list items

I then went back to the Logic App and the **Logic app designer** blade, and added a new action step.

- Connector: `HTTP`
- Method: `GET`
- URI: `https://graph.microsoft.com/v1.0/sites/<tenant>.sharepoint.com:/sites/LBDM365Automation:/lists/OrderList/items?$select=id,webUrl,fields,createdDateTime&$expand=fields($select=Title,Owner,Description,AutomationCompleted)&$top=999`
- Authentication
    - Authentication type: `Managed Identity`
    - Managed identity: `System-assigned managed identity`
    - Audience: `https://graph.microsoft.com`

I saved the new configuration and triggered the Logic App. And behold - status code `200` and a response body with the list items!

![Graph Explorer Get List Items](/assets/img/posts/2022-02-14/logicapp-http-action-result.png)

Success! The Logic App is able to work with data in SharePoint Online sites authenticating with its least-privileged Managed Identity, but only for sites it is specifically granted app access to. To verify, I created a new Team named **LBD M365 Automation 2**, added a SharePoint list named **OrderList** to the SharePoint site with a few list items. Using [Graph Explorer](https://learningbydoing.cloud/blog/getting-started-with-microsoft-graph-part3/#what-is-graph-explorer) I verified that I was able to retrieve the items in the SharePoint List using my own account.

```text
GET https://graph.microsoft.com/v1.0/sites/<tenant>.sharepoint.com:/sites/LBDM365Automation2:/lists/OrderList/items?$select=id,webUrl,fields,createdDateTime&$expand=fields($select=Title,Owner,Description,AutomationCompleted)
```

And then, when updating the `URI` in the HTTP connector in the Logic App to retrieve the list items from the _new_ SharePoint site - which the Managed Identity has not been granted app access to - it resulted in status code `403` with the message `Access denied`. Just as expected.

Now you know how to utilize `Sites.Selected` application scope and app access roles in SharePoint Online to grant least-privileged access for automation processes utilizing Managed Identities.

And that concludes this blog post, thanks for reading!

Be sure to provide any feedback on [Twitter](https://twitter.com/stianstrysse/status/1491130326252670976) or [LinkedIn](https://www.linkedin.com/posts/stianstrysse_stop-using-client-secrets-and-certificates-activity-6896896311590039552-Ofp6).
