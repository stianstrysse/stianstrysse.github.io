---
layout: post
title: Control access to Azure Storage Blobs with Attribute-based Access Control conditions
subtitle: Let's look at how to configure access to Azure Storage Blobs using Attribute-based Access Control (ABAC) paired with Custom Security Attributes
categories: AZUREAD AZURE IDENTITY GOVERNANCE ABAC RBAC
thumbnail-img: /assets/img/posts/2022-01-15/az-abac-conditions-thumbnail.png
author: Stian A. Strysse
---

Last week I published a [blog post describing the basics of Custom Security Attributes](https://learningbydoing.cloud/blog/getting-started-with-custom-security-attributes-in-azuread/), and how it can be utilized paired with ABAC. Now I will dive further into this topic and describe how to get a working configuration with ABAC conditions using Custom Security Attributes for [Azure Blobs Storage](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blobs-introduction).

![Azure ABAC conditions thumbnail](/assets/img/posts/2022-01-15/az-abac-conditions-thumbnail.png)

Blog post contents:

+ [What is Azure ABAC?](#what-is-azure-abac)
+ [Configure Custom Security Attribute](#configure-custom-security-attribute)
+ [Configure Azure Storage blob access with ABAC conditions](#configure-azure-storage-blob-access-with-abac-conditions)
+ [Test the configuration](#test-the-configuration)

### What is Azure ABAC?

*[Azure ABAC](https://docs.microsoft.com/en-us/azure/role-based-access-control/conditions-overview)* is short for *Attribute-based Access Control*. This is a new feature introduced by Microsoft for Azure Blob Storage and is in [public preview](https://techcommunity.microsoft.com/t5/azure-active-directory-identity/introducing-attribute-based-access-control-abac-in-azure/ba-p/2147069). ABAC brings a new flavor to access control management of Azure resources, and even though only Azure Blob Storage is supported right now ABAC will likely be introduced to a lot of other Azure resources in the future - to solve access scenarios that are complicated or hard to manage with regular [Azure RBAC](https://docs.microsoft.com/en-us/azure/role-based-access-control/overview) (short for Role-based Access Control).

In the [Introducing Azure AD custom security attributes](https://techcommunity.microsoft.com/t5/azure-active-directory-identity/introducing-azure-ad-custom-security-attributes/ba-p/2147068) article posted in the Azure Active Directory blog on December 1, 2021, Microsoft explains that Custom Security Attributes builds on the ABAC public preview. Now you can grant users access to Azure Blob Storage with [Attribute-based Access Control (ABAC)](https://docs.microsoft.com/en-us/azure/role-based-access-control/conditions-overview) utilizing Custom Security Attributes instead of just through Role-Based Access Control (RBAC), which means fine-grained access control with fewer Azure role assignments.

So, let's try out this public preview feature!

### Configure Custom Security Attribute

We will need a Custom Security Attribute in the tenant for this purpose. See my [blogpost](https://learningbydoing.cloud/blog/getting-started-with-custom-security-attributes-in-azuread/) for details on how to create such attributes, what permissions are needed to do such operations etc.

{: .box-note}
**Note**: Once you create an Attribute Set, it cannot be deleted from the tenant nor have its Attribute Set name changed. Always test new stuff in a demo tenant!

First I create a new Attribute Set named `ProjectData` by using Graph Explorer, see my [blogpost](https://learningbydoing.cloud/blog/getting-started-with-microsoft-graph-part3/#working-with-graph-explorer) on how to get started.

1. Go to [Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer) and sign in to the tenant.
2. Consent to Graph permissions `CustomSecAttributeAssignment.ReadWrite.All` and `CustomSecAttributeDefinition.ReadWrite.All`.
3. Make a `POST` request to endpoint `/directory/attributeSets` (beta) with the properties `id` (Attribute Set name), `description` and `maxAttributesPerSet` in the `body`:

    ```json
    {
        "id": "ProjectData",
        "description": "Set contains all security attributes for ProjectData",
        "maxAttributesPerSet": 25
    }
    ```

    The Graph response will be:

    ```json
    {
        "@odata.context": "https://graph.microsoft.com/beta/$metadata#directory/attributeSets/$entity",
        "description": "Set contains all security attributes for ProjectData",
        "id": "ProjectData",
        "maxAttributesPerSet": 25
    }
    ```

See the [Graph documentation for attributeSet resource type](https://docs.microsoft.com/en-us/graph/api/resources/attributeset?view=graph-rest-beta) on Microsoft Docs for more details.

Next, I create a Custom Security Attribute (string) named `ProjectOrganization` within the newly created Attribute Set.

{: .box-note}
**Note**: Once you create a Custom Security Attribute, it cannot be deleted from the tenant nor have its Attribute name changed. It can however be disabled if no longer in use.

1. Go to [Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer) and sign in to the tenant.
2. Make a `POST` request to endpoint `/directory/customSecurityAttributeDefinitions` (beta) with the properties `attributeSet` (Attribute Set name), `description`, `name` (Custom Security Attribute name), `status` (whether the attribute is enabled or disabled), `type` (data type), `isCollection` (if attribute is multivalue), `usePreDefinedValuesOnly` (whether only allowing predefined values to be assigned) and `isSearchable` (whether attribute values are indexed) in the `body`:

    ```json
    {
        "attributeSet":"ProjectData",
        "description":"ProjectOrganization code that the user is a member of",
        "isCollection":false,
        "isSearchable":true,
        "name":"ProjectOrganization",
        "status":"Available",
        "type":"String",
        "usePreDefinedValuesOnly": false
    }
    ```

    The Graph response will be: 

    ```json
    {
        "@odata.context": "https://graph.microsoft.com/beta/$metadata#directory/customSecurityAttributeDefinitions/$entity",
        "attributeSet": "ProjectData",
        "description": "ProjectOrganization code that the user is a member of",
        "id": "ProjectData_ProjectOrganization",
        "isCollection": false,
        "isSearchable": true,
        "name": "ProjectOrganization",
        "status": "Available",
        "type": "String",
        "usePreDefinedValuesOnly": false
    }
    ```

See the [Graph documentation for customSecurityAttributeDefinition resource type](https://docs.microsoft.com/en-us/graph/api/resources/customsecurityattributedefinition?view=graph-rest-beta) on Microsoft Docs for more details.

And lastly, we need to assign the Custom Security Attribute with a value to a user which we will test with in the next chapter.

1. Go to [Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer) and sign in to the tenant.
2. Make a `PATCH` request to endpoint `/users/{id|upn}` (beta) with the correct user `id` or `userPrincipalName`. Add the properties in the `body` as shown below, `ProjectData` is the Attribute Set name and `ProjectOrganization` is the Custom Security Attribute name, while `dpt-23184` is the Attribute value:

    ```json
    {
        "customSecurityAttributes": {
            "ProjectData": {
                "@odata.type": "#Microsoft.DirectoryServices.CustomSecurityAttributeValue",
                "ProjectOrganization": "dpt-23184"
            }
        }
    }
    ```

    The Graph response will be: 

    ```http
    HTTP/1.1 204 No Content
    ```

To verify the assigned Custom Security Attributes on the user, make a `GET` request to endpoint `/users/{id|upn}?$select=id,userPrincipalName,customSecurityAttributes` (beta) with the correct user `id` or `userPrincipalName`. The Graph response will be:

```json
{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#users(id,userPrincipalName,customSecurityAttributes)/$entity",
    "id": "26406275-67f9-47fe-a80b-f71b18865cf2",
    "userPrincipalName": "user@tenant.onmicrosoft.com",
    "customSecurityAttributes": {
        "ProjectData": {
            "@odata.type": "#microsoft.graph.customSecurityAttributeValue",
            "ProjectOrganization": "dpt-23184"
        }
    }
}
```

This example was for assigning a `string` Attribute. See the [Graph documentation for assigning, updating or removing Custom Security Attributes](https://docs.microsoft.com/en-us/graph/custom-security-attributes-examples) on Microsoft Docs for other data types and additional information.

### Configure Azure Storage blob access with ABAC conditions

The prerequisites are in place and I can go ahead and create a new Azure Storage Account to play around with, and set up RBAC and ABAC.

1. Go to the [Azure Portal](https://portal.azure.com) and create a new Storage Account.
2. Once deployed, visit the **Access Control (IAM)** blade on the Storage Account, go to the **Role assignments** tab.
3. Click **+ Add** -> **Add role assignment**, on the **Members** tab select `Reader` and click **Next**, specify the user that was given the Custom Security Attribute earlier, and click **Review + assign**.

    _Note: The `Reader` RBAC role is only granted to the user so they will be able to actually see the Storage Account in the Azure Portal. This role does not grant any access to the data within the Storage Account._

4. Then click **+ Add** -> **Add role assignment** again, on the **Members** tab select `Storage Blob Data Reader` and click **Next**, specify the user that was given the Custom Security Attribute earlier (or a group that the user is a member of), click **Next**.
5. On the `Conditions` tab, click **+ Add condition** and use the visual editor to configure conditions as displayed below:

    ![Azure Storage Blob ABAC](/assets/img/posts/2022-01-15/storage-blob-abac.png)
    
    If you prefer to use code editor instead, use:

    ```text
    (
     (
      !(ActionMatches{'Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read'} AND SubOperationMatches{'Blob.Read.WithTagConditions'})
     )
     OR 
     (
      @Principal[Microsoft.Directory/CustomSecurityAttributes/Id:ProjectData_ProjectOrganization] StringEquals @Resource[Microsoft.Storage/storageAccounts/blobServices/containers/blobs/tags:ProjectOrgTag<$key_case_sensitive$>]
     )
    )
    ```

6. Click **Review + assign** to save the new role assignment with conditions.

I have now configured an Azure RBAC role assignment with ABAC conditions for Storage Blob index tags. There are other ways to configure ABAC, e.g. matching name on containers within the Storage Account - visit [Microsoft Docs](https://docs.microsoft.com/en-us/azure/storage/common/storage-auth-abac-examples?toc=/azure/storage/blobs/toc.json) for more examples on how to configure ABAC.

### Test the configuration

In order to test the configuration I first need to upload some blobs with index tags in a container within the Storage Account. I have prepared 3 text files with random content on my local machine.

1. Go to the Storage Account created earlier in the [Azure Portal](https://portal.azure.com), visit the **Containers** blade.
2. Create a new private container, then open it.
3. Click **Upload**, select the first file and click **Advanced** - make sure to add the correct `Blob index tag key` and `value`. 

    _Note: The `key` value is the same key as configured in the ABAC conditions (step 5), while `value` is the same value as set in the Custom Security Attribute on the user._

    ![Azure Storage Blob upload file](/assets/img/posts/2022-01-15/storage-blob-uploadfile.png)

4. Then upload the two other files - but this time set no value or some other Blob index tag value, meaning that the user should not be able to read these blobs.

    ![Azure Storage Blob uploaded files](/assets/img/posts/2022-01-15/storage-blob-uploadedfiles.png)

Now it's time to log in to the Azure Portal with the test user to verify that ABAC conditions actually works. 

1. Log in to the [Azure Portal](https://portal.azure.com) as the test user that was given the Customer Security Attribute earlier.
2. Go to the Storage Account, then **Containers** blade and click on the container. Make sure that **Authentication method** is set to `Azure AD User Account`.

The user should now see the 3 blobs that were uploaded. However, the user should **only** be able to click and read the contents of the blob that has the same `Blob index tag value` as the user's Custom Security Attribute value, as the `Storage Blob Data Reader` role assignment with ABAC conditions is only valid for blobs where the index tag matches. When the user tries to click and read the other two blobs it will result in a `No Access` error:

![Azure Storage Blob ABAC condition error](/assets/img/posts/2022-01-15/storage-blob-abac-condition-error.png)

Pretty cool! This means I can have thousands of blobs within one or more Storage Accounts, and by granting exactly **one** RBAC role assignment with ABAC conditions I can control which blobs users can access based on a value tagged in their Custom Security Attributes. This is just one scenario showcasing Azure ABAC conditions using blob index tags. As mentioned earlier, there are other ways to configure ABAC conditions, both with and without using Custom Security Attributes - visit [Microsoft Docs](https://docs.microsoft.com/en-us/azure/storage/common/storage-auth-abac-examples?toc=/azure/storage/blobs/toc.json) for more examples on how to configure ABAC. 

I believe and hope that we will see a lot more support for ABAC and Custom Security Attributes built into Azure and Azure AD services in the near future. Stay tuned!

And that concludes this blogpost, thanks for reading!

Be sure to provide any feedback on [Twitter](https://twitter.com/stianstrysse/status/1482325535703379969) or [LinkedIn](https://www.linkedin.com/posts/stianstrysse_control-access-to-azure-storage-blobs-with-activity-6888090844876832768-mxFq).