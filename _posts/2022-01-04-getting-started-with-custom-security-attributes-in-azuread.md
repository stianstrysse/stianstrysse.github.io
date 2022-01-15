---
layout: post
title: Getting started with Custom Security Attributes in Azure AD
subtitle: This blogpost explores the new Custom Security Attributes public preview feature in Azure AD
categories: AZUREAD AZURE IDENTITY GOVERNANCE ABAC RBAC
thumbnail-img: /assets/img/posts/2022-01-04/aad-custsecattribs-thumbnail.png
author: Stian A. Strysse
---

Azure AD has a schema with common attributes for resources like users, e.g. `displayName`, `userPrincipalName`, `companyName`, `department` and so on. You can also [add custom extension attributes via an Application object](https://learningbydoing.cloud/blog/getting-started-with-azuread-extension-attributes/) to extend the schema. However, these attributes are public for all Azure AD users in the organization and should never contain sensitive information. But, what if you need to set a HR termination date on Azure AD users to be consumed by an automation process, or set other sensitive attributes which should only be visible for certain individuals in the organization? And what if you want to control access to Azure resources using attribute values?

![AAD Custom Sec thumb](/assets/img/posts/2022-01-04/aad-custsecattribs-thumbnail.png)

You now have a way to solve this with Microsoft's public preview of [Custom Security Attributes](https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/custom-security-attributes-overview) in Azure AD. Custom Security Attributes are organization-specific key-value paired attributes that can be assigned to Azure AD objects like Users, Service Principals and Managed Identities. This type of attribute is by default not available to anyone, not even Global Admins, without specifically assigning permissions.

License requirement for Custom Security Attributes is *Azure AD Premium P1*.

+ [Assign Azure AD roles for managing Custom Security Attributes, Sets, assignments and values](#assign-azure-ad-roles-for-managing-custom-security-attributes-sets-assignments-and-values)
+ [Create an Attribute Set](#create-an-attribute-set)
+ [Create Custom Security Attributes](#create-custom-security-attributes)
+ [Assign Custom Security Attribute to a user](#assign-custom-security-attribute-to-a-user)
+ [Control Azure Storage Blob access with ABAC](#control-azure-storage-blob-access-with-abac)
+ [The future of Custom Security Attributes](#the-future-of-custom-security-attributes)
+ [Known issues with Custom Security Attributes](#known-issues-with-custom-security-attributes)

Let's dive into this functionality to see how it's set up and how it can be used.

### Assign Azure AD roles for managing Custom Security Attributes, Sets, assignments and values

Custom Security Attributes are created within an Attribute Set, and each Attribute Set has its own access control for defining who can read, define or assign Custom Security Attributes for that specific Set. However, built-in Azure AD roles can be assigned to administrators for providing access to **all** Custom Security Attributes, Sets, assignments and values tenant-wide. Global Admin and other high-privileged roles do not have access to Custom Security Attributes by default.

The following built-in roles exists both for tenant-level and for specific Attribute Sets:

| Role | Permissions (can do) | Restrictions (cannot do) |
|---|---|---|
| [Attribute Definition Reader](https://docs.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#attribute-definition-reader) | ✔️ *Read* Attribute Set <br />✔️ *Read* Attribute definition | ❌ *Manage* Attribute Set <br />❌ *Manage* Attribute definition <br />❌ *Read / assign / manage* Attribute values |
| [Attribute Definition Administrator](https://docs.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#attribute-definition-administrator) | ✔️ *Manage* Attribute Set <br />✔️ *Manage* Attribute definition | ❌ *Read / assign / manage* Attribute values |
| [Attribute Assignment Reader](https://docs.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#attribute-assignment-reader) | ✔️ *Read* Attribute Set <br />✔️ *Read* Attribute definition <br />✔️ *Read* Attribute values | ❌ *Manage* Attribute Set <br />❌ *Manage* Attribute definition <br />❌ *Assign / manage* Attribute values |
| [Attribute Assignment Administrator](https://docs.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#attribute-assignment-administrator) | ✔️ *Read* Attribute Set <br />✔️ *Read* Attribute definition <br />✔️ *Assign / manage* Attribute values | ❌ *Manage* Attribute Set <br />❌ *Manage* Attribute definition |

This means that for an administrator to be able to manage all Custom Security Attributes, Sets, assignments and values tenant-wide, they need to be assigned both **Attribute Definition Administrator** and **Attribute Assignment Administrator** Azure AD roles at the tenant-level (through the [Azure AD Portal: Roles and administrators blade](https://aad.portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/RolesAndAdministrators) or [PIM](https://aka.ms/pim)). An individual who only need specific permissions to specific Custom Security Attributes should be assigned set-level role(s) directly on the Attribute Set(s) ([AAD: Custom Security Attributes blade](https://aad.portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/CustomAttributesCatalog) -> Attribute Set -> Roles and administrators).

[Microsoft Docs](https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/custom-security-attributes-manage) has more information on this topic.

There's also tenant-wide Graph scopes available granting permissions to manage Custom Security Attributes, Attribute Sets, assignments and values through Microsoft Graph. 

- CustomSecAttributeAssignment.ReadWrite.All
- CustomSecAttributeDefinition.ReadWrite.All

At the time of writing there are only Graph scopes available for read/write operations, not for read-only.

### Create an Attribute Set

After assigning tenant-level permissions, the next thing we need to do is to create an Attribute Set. This can be done either in the Azure AD Portal or via Microsoft Graph. 

{: .box-note}
**Note**: Once you create an Attribute Set, it cannot be deleted from the tenant nor have its Attribute Set name changed. Always test new stuff in a demo tenant!

**Azure AD Portal**: 

1. Go to the [Azure AD Portal: Custom Security Attributes blade](https://aad.portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/CustomAttributesCatalog).
2. Click **Add attribute set** and specify `Attribute set name` and `Description`.

    ![AAD Custom Sec Attributes - create](/assets/img/posts/2022-01-04/aad-customsecattribs-createset.png)

3. Click **Add** to create the new Attribute Set in the tenant.

    ![AAD Custom Sec Attributes - created](/assets/img/posts/2022-01-04/aad-customsecattribs-createdset.png)

**Microsoft Graph** (by using Graph Explorer, see my [blogpost](https://learningbydoing.cloud/blog/getting-started-with-microsoft-graph-part3/#working-with-graph-explorer) on how to get started):

1. Go to [Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer) and sign in to your tenant
2. Consent to Graph permissions `CustomSecAttributeAssignment.ReadWrite.All` and `CustomSecAttributeDefinition.ReadWrite.All`
3. Make a `POST` request to endpoint `/directory/attributeSets` (beta) with the properties `id` (Attribute Set name), `description` and `maxAttributesPerSet` in the `body`:

    ```json
    {
        "id": "BusinessProjects",
        "description": "Set contains all security attributes for BusinessProjects",
        "maxAttributesPerSet": 25
    }
    ```

    The Graph response will be:

    ```json
    {
        "@odata.context": "https://graph.microsoft.com/beta/$metadata#directory/attributeSets/$entity",
        "description": "Set contains all security attributes for BusinessProjects",
        "id": "BusinessProjects",
        "maxAttributesPerSet": 25
    }
    ```

See the [Graph documentation for attributeSet resource type](https://docs.microsoft.com/en-us/graph/api/resources/attributeset?view=graph-rest-beta) on Microsoft Docs for more details.

### Create Custom Security Attributes

Next up is to create one or more Custom Security Attributes in our new Attribute Set. Custom Security Attributes support `string`, `integer` and `boolean` data types - and with exception for `boolean` it also supports multivalue. There's also an option for only allowing predefined values to be assigned. Creation of Custom Security Attributes can be done either in the Azure AD Portal or via Microsoft Graph.

{: .box-note}
**Note**: Once you create a Custom Security Attribute, it cannot be deleted from the tenant nor have its Attribute name changed. It can however be disabled if no longer in use.

**Azure AD Portal**: 

1. Go to the [Azure AD Portal: Custom Security Attributes blade](https://aad.portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/CustomAttributesCatalog) and click on the Attribute Set created earlier.
2. Click **Add attribute** and specify `Attribute name`, `Description`, `Data type`, and choose whether to allow multiple values or require predefined values only.

    ![AAD Custom Sec Attributes - create attribute](/assets/img/posts/2022-01-04/aad-customsecattribs-createattribute.png)

3. Click **Add** to create the new Custom Security Attribute in the Attribute Set.

    ![AAD Custom Sec Attributes - created attribute](/assets/img/posts/2022-01-04/aad-customsecattribs-createdattribute.png)

**Microsoft Graph**:

1. Go to [Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer) and sign in to your tenant
2. Make a `POST` request to endpoint `/directory/customSecurityAttributeDefinitions` (beta) with the properties `attributeSet` (Attribute Set name), `description`, `name` (Custom Security Attribute name), `status` (whether the attribute is enabled or disabled), `type` (data type), `isCollection` (if attribute is multivalue), `usePreDefinedValuesOnly` (whether only allowing predefined values to be assigned) and `isSearchable` (whether attribute values are indexed) in the `body`:

    ```json
    {
        "attributeSet":"BusinessProjects",
        "description":"ProjectID for the assigned BusinessProject",
        "isCollection":false,
        "isSearchable":true,
        "name":"ProjectID",
        "status":"Available",
        "type":"String",
        "usePreDefinedValuesOnly": false
    }
    ```

    The Graph response will be: 

    ```json
    {
        "@odata.context": "https://graph.microsoft.com/beta/$metadata#directory/customSecurityAttributeDefinitions/$entity",
        "attributeSet": "BusinessProjects",
        "description": "ProjectID for the assigned BusinessProject",
        "id": "BusinessProjects_ProjectID",
        "isCollection": false,
        "isSearchable": true,
        "name": "ProjectID",
        "status": "Available",
        "type": "String",
        "usePreDefinedValuesOnly": false
    }
    ```

See the [Graph documentation for customSecurityAttributeDefinition resource type](https://docs.microsoft.com/en-us/graph/api/resources/customsecurityattributedefinition?view=graph-rest-beta) on Microsoft Docs for more details.

### Assign Custom Security Attribute to a user

The prerequisites are complete and we can now assign the new Custom Security Attribute to a user, which can be done either in the Azure AD Portal or via Microsoft Graph.

**Azure AD Portal**: 

1. Go to the [Azure AD Portal: Users blade](https://aad.portal.azure.com/#blade/Microsoft_AAD_IAM/UsersManagementMenuBlade/MsGraphUsers) and click on a user.
2. Within the user's profile, click **Custom security attributes**, then click **Add assignment**.
3. In the `Attribute set` dropdown, select the Attribute Set - and in the `Attribute name` dropdown, select the Custom Security Attribute created earlier. Then type in an Attribute value in the `Assigned value` column.

    ![AAD Custom Sec Attributes - assign attribute](/assets/img/posts/2022-01-04/aad-customsecattribs-assignattribute.png)

4. Click **Save** to assign the Custom Security Attribute and value to the user.

**Microsoft Graph**:

1. Go to [Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer) and sign in to your tenant
2. Make a `PATCH` request to endpoint `/users/{id|upn}` (beta) with the correct user `id` or `userPrincipalName`. Add the properties in the `body` as shown below, `BusinessProjects` is the Attribute Set name and `ProjectID` is the Custom Security Attribute name, while `PBX4718` is the Attribute value:

    ```json
    {
        "customSecurityAttributes": {
            "BusinessProjects": {
                "@odata.type": "#Microsoft.DirectoryServices.CustomSecurityAttributeValue",
                "ProjectID": "PBX4718"
            }
        }
    }
    ```

    The Graph response will be: 

    ```http
    HTTP/1.1 204 No Content
    ```

    To view the assigned Custom Security Attributes on a user, make a `GET` request to endpoint `/users/{id|upn}?$select=id,userPrincipalName,customSecurityAttributes` (beta) with the correct user `id` or `userPrincipalName`. The Graph response will be:

    ```json
    {
        "@odata.context": "https://graph.microsoft.com/beta/$metadata#users(id,userPrincipalName,customSecurityAttributes)/$entity",
        "id": "26406275-67f9-47fe-a80b-f71b18865cf2",
        "userPrincipalName": "user@tenant.onmicrosoft.com",
        "customSecurityAttributes": {
            "BusinessProjects": {
                "@odata.type": "#microsoft.graph.customSecurityAttributeValue",
                "ProjectID": "PBX4718"
            }
        }
    }
    ```

This example was for assigning a `string` Attribute. See the [Graph documentation for assigning, updating or removing Custom Security Attributes](https://docs.microsoft.com/en-us/graph/custom-security-attributes-examples) on Microsoft Docs for other data types and additional information.

### Control Azure Storage Blob access with ABAC

*ABAC* is short for *Attribute-based Access Control*. This is a new feature introduced by Microsoft for Azure Blob Storage and is now in [public preview](https://techcommunity.microsoft.com/t5/azure-active-directory-identity/introducing-attribute-based-access-control-abac-in-azure/ba-p/2147069). ABAC brings a new flavor to access control management of Azure resources, and even though only Azure Blob Storage is supported right now ABAC will likely be introduced to a lot of other Azure resources in the future.

In the [Introducing Azure AD custom security attributes](https://techcommunity.microsoft.com/t5/azure-active-directory-identity/introducing-azure-ad-custom-security-attributes/ba-p/2147068) article posted in the Azure Active Directory blog on December 1, 2021, Microsoft explains that Custom Security Attributes builds on the ABAC public preview. Now you can grant users access to Azure Blob Storage with [Attribute-based Access Control (ABAC)](https://docs.microsoft.com/en-us/azure/role-based-access-control/conditions-overview) utilizing Custom Security Attributes instead of just through Role-Based Access Control (RBAC), which means fine-grained access control with fewer Azure role assignments.

Check out my blog post [Control access to Azure Storage Blobs with Attribute-based Access Control conditions](https://learningbydoing.cloud/blog/control-access-to-azure-storage-blobs-with-abac/) to see how this works.

### The future of Custom Security Attributes

Having a way of extending the Azure AD schema for Users, Service Principals and Managed Identities without having to rely on app registrations is really great - and it's even better that access to these attributes can be controlled with RBAC and that no users have default access to these. Right now you can assign Custom Security Attributes and consume them in apps and scripts by using Microsoft Graph, and we will likely see that more services in Azure and Azure AD will be able to consume these in the future.

I really hope and expect that Microsoft quickly invests further into Custom Security Attributes to help customers solve a broad number of scenarios, like supporting dynamic membership rules for Azure AD groups, Access Package assignments, SAML and OpenID Connect claims, Conditional Access policies and more around access to Azure resources with Attribute-Based Access Control (ABAC). I'll be sure to update my blog as they announce new features.

### Known issues with Custom Security Attributes

At the time of writing there are some [known issues listed on Microsoft Docs](https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/custom-security-attributes-overview#known-issues) regarding Custom Security Attributes that you should be aware of. Remember that this feature is in public preview and that Microsoft is still working on making the feature complete and ready for general availability. 

In my view the most important known issue is that Global Admins can read Azure AD audit logs to see Custom Security Attribute definitions and assignments, including Attribute values. If you export audit logs elsewhere then others might also be able to view these. Also, users with attribute set-level role assignments can view other Attribute Sets and Custom Security Attribute definitions. And the current lack of read-only Graph scope is also a downside. So you might want to wait with adding highly sensitive attribute values until the feature goes GA.

And that concludes this blogpost, thanks for reading!

Be sure to provide any feedback on [Twitter](https://twitter.com/stianstrysse/status/1478337726948810752) or [LinkedIn](https://www.linkedin.com/posts/stianstrysse_getting-started-with-custom-security-attributes-activity-6884103665645318144-YHC_).
