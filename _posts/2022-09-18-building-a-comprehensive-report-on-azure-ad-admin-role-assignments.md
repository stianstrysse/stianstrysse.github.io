---
layout: post
title: Building a comprehensive report on Azure AD admin role assignments in Powershell
subtitle: Keeping an eye on Azure AD administrative role assignments is crucial for tenant security and compliance. Forget about the built-in PIM report in the Azure AD portal - take reporting to the next level and build your own report with Graph, KQL and Powershell.
thumbnail-img: /assets/img/posts/2022-09-18/shareimg-unsplash.jpg
categories: AZUREAD AZURE IDENTITY GOVERNANCE GRAPH KQL POWERSHELL MICROSOFTGRAPH ADMINISTRATIVEROLES AUDIT
author: Stian A. Strysse
---

Unassigning inactive roles, verifying that all role holders have registered MFA and are active users, auditing service principals, role-assignable groups and guests with roles, move users from active to eligible roles in PIM ([Privileged Identity Management](https://docs.microsoft.com/en-us/azure/active-directory/privileged-identity-management/)), and making sure that no synchronized users have privileged roles are just a few ideas for why you should be reporting on this topic.

In this blogpost I will showcase how to gather data from various sources and compile it all into an actionable status report. Since different tenants have different needs and ways of working, I'm providing examples so that you can write your own custom-tailored script.

The report will list the following records:

1. Users with eligible or active Azure AD admin roles - including details on last role activation date, role assignment and expiration dates, MFA status and last sign-in date, admin owner account status etc.
2. Service Principals / Applications and Managed Identities with active Azure AD admin roles - including details on last authentication date, tenant ownership, etc.
3. Role-assignable groups with eligible or active Azure AD admin roles

{: .box-note}
**Note**: Role-assignable groups granted one or more Azure AD admin roles will be listed in the report but users with active or eligible membership to such groups will currently not be listed.

See the [Report examples](#report-examples) chapter for details.

* [Prerequisites](#prerequisites)
* [Connecting to Graph and Log Analytics](#connecting-to-graph-and-log-analytics)
* [Extracting data](#extracting-data)
  * [MFA registration details](#mfa-registration-details)
  * [Role assignments](#role-assignments)
  * [Principal last sign-in date](#principal-last-sign-in-date)
  * [Eligible role last activation date](#eligible-role-last-activation-date)
  * [Default MFA method and capability](#default-mfa-method-and-capability)
  * [Admin account owner](#admin-account-owner)
  * [Service Principal owner organization](#service-principal-owner-organization)
* [Compiling the report](#compiling-the-report)
  * [Report examples](#report-examples)
  * [Example script](#example-script)

## Prerequisites

These Powershell modules are required:

* [Graph Powershell SDK](https://docs.microsoft.com/en-us/powershell/microsoftgraph/installation?view=graph-powershell-1.0)
* [Azure Powershell](https://docs.microsoft.com/en-us/powershell/azure/install-az-ps?view=azps-8.2.0)

Other prerequisites:

* Global Reader role (or other AAD roles granting enough read-access)
* Admin consent to any required non-consented Graph scopes (read-only) in Graph Powershell SDK.
* Reader-role on the Log Analytics workspace where the Azure AD `Sign-in` and `Audit` logs are exported.

## Connecting to Graph and Log Analytics

Connect to Graph with the Graph Powershell SDK using the required read-only scopes, and select the `beta` endpoint as required by some of the cmdlets:

```powershell
Connect-MgGraph -Scopes RoleEligibilitySchedule.Read.Directory, RoleAssignmentSchedule.Read.Directory, CrossTenantInformation.ReadBasic.All, AuditLog.Read.All, User.Read.All
Select-MgProfile -Name Beta
```

Then connect to Azure with the Azure Powershell module, for running KQL queries on the Log Analytics workspace data. Read my [Query Azure AD logs with KQL from Powershell](https://learningbydoing.cloud/blog/query-log-analytics-with-kql-from-powershell/) blogpost for more information on running KQL queries in Powershell. Update the various parameters according to your environment.

```powershell
Connect-AzAccount
Set-AzContext -Subscription "my-subscription"
$workspaceName = "p-aadlogs-loganalyticsworkspace"
$workspaceRG = "p-aadlogs-loganalytics"
$workspaceID = (Get-AzOperationalInsightsWorkspace -Name $workspaceName -ResourceGroupName $workspaceRG).CustomerID
```

## Extracting data

We need to extract data from various sources using Microsoft Graph and KQL queries in Log Analytics.

### MFA registration details

To report on MFA registration details for Azure AD admin role holders it is likely most efficient to extract all registration details and create a [hashtable](https://docs.microsoft.com/en-us/powershell/scripting/learn/deep-dives/everything-about-hashtable?view=powershell-7.2#group-object--ashashtable) for quick lookup, depending on the number of users in the tenant.

```powershell
# Get MFA registration details
# Graph API: https://graph.microsoft.com/beta/reports/authenticationMethods/userRegistrationDetails
Write-Host -ForegroundColor Yellow "Fetching MFA registration details report"
$mfaRegistrationDetails = Get-MgReportAuthenticationMethodUserRegistrationDetail -All:$true
$mfaRegistrationDetailsHashmap = $mfaRegistrationDetails | Group-Object -Property Id -AsHashTable
Write-Host -ForegroundColor Yellow "Found $($mfaRegistrationDetails.count) MFA registration detail records"
```

### Role assignments

Assigned roles are active role assignments. This query will also return eligible role assignments which are currently activated through PIM, so we'll filter those out as they will just be duplicates in the report as they are also listed as eligible roles.

```powershell
# Get assigned role assignments
Write-Host -ForegroundColor Yellow "Fetching assigned role assignments, might take a minute..."
$assignedRoleAssignments = Get-MgRoleManagementDirectoryRoleAssignmentScheduleInstance -ExpandProperty "*" -All:$true
$activatedRoleAssignments = $assignedRoleAssignments | Where-Object { $_.AssignmentType -eq 'Activated' }
$filteredAssignedRoleAssignments = $assignedRoleAssignments | Where-Object { $_.AssignmentType -eq 'Assigned' }
Write-Host -ForegroundColor Yellow "Found $($filteredAssignedRoleAssignments.count) assigned role assignments"
```

Eligible roles are role assignments requiring activation in PIM.

```powershell
# Get eligible role assignments
Write-Host -ForegroundColor Yellow "Fetching eligible role assignments, might take a minute..."
$eligibleRoleAssignments = Get-MgRoleManagementDirectoryRoleEligibilitySchedule -ExpandProperty "*" -All:$true
Write-Host -ForegroundColor Yellow "Found $($eligibleRoleAssignments.count) eligible PIM role assignments, whereof $($activatedRoleAssignments.count) are activated"
```

Then we combine the two assignment types into one array. Use the `Select-Object` cmdlet to pick out a few records while developing and testing the script.

```powershell
# Combine assignments
$allRoleAssignments = @(
    $eligibleRoleAssignments #| Select-Object -First 10
    $filteredAssignedRoleAssignments #| Select-Object -First 10
)
```

Now we have all the assignment objects we need in the `$allRoleAssignments` array, and will process each of those objects in a `foreach` loop to fetch other necessary data. In the following examples I've populated the `$roleObject` variable with one object from the `$allRoleAssignments` array.

Since the `$allRoleAssignments` array may contain both users and Service Principals with active or eligible role assignments, the `$roleObject.Principal.AdditionalProperties.'@odata.type` property will tell which principal type the current object is - either `'#microsoft.graph.user` or `#microsoft.graph.servicePrincipal`. And for Service Principals we can differentiate on types in the `$roleObject.Principal.AdditionalProperties.servicePrincipalType` property - which is either `Application` or `ManagedIdentity`.

### Principal last sign-in date

The quickest way to get an Azure AD user's last sign-in date is to query Graph for the user and selecting `signInActivity`.

```powershell
$principalLastSignIn = $null
$principalSignInActivity = Invoke-MgGraphRequest -Method GET -Uri "https://graph.microsoft.com/beta/users/$($roleObject.Principal.Id)?`$select=id,userPrincipalName,userType,signInActivity"
if($principalSignInActivity) {
    if($principalSignInActivity.signInActivity.lastSignInDateTime -gt $principalSignInActivity.signInActivity.lastNonInteractiveSignInDateTime) {
        $principalLastSignIn = $principalSignInActivity.signInActivity.lastSignInDateTime
    } else { $principalLastSignIn = $principalSignInActivity.signInActivity.lastNonInteractiveSignInDateTime }
}
```

For Service Principals we need to query the Azure AD logs in Log Analytics with KQL to fetch the date when the Service Principal last signed in.

KQL query for Service Principal of type `Application`:

```powershell
# KQL query for SP last sign-in
$principalLastSignIn = $null
$query = "AADServicePrincipalSignInLogs
| where ResultType == '0'
| where TimeGenerated > ago(90d)
| where AppId == '$($roleObject.Principal.AdditionalProperties.appId)'
| sort by TimeGenerated desc
| limit 1"

$kqlQuery = Invoke-AzOperationalInsightsQuery -WorkspaceId $WorkspaceID -Query $query
$principalLastSignIn = $kqlQuery.Results.TimeGenerated
```

KQL query for Service Principals of type `ManagedIdentity`:

```powershell
# KQL query for MSI last sign-in
$principalLastSignIn = $null
$query = "AADManagedIdentitySignInLogs
| where ResultType == '0'
| where TimeGenerated > ago(90d)
| where AppId == '$($roleObject.Principal.AdditionalProperties.appId)'
| sort by TimeGenerated desc
| limit 1"

$kqlQuery = Invoke-AzOperationalInsightsQuery -WorkspaceId $WorkspaceID -Query $query
$principalLastSignIn = $kqlQuery.Results.TimeGenerated
```

### Eligible role last activation date

We also need to fetch the latest date of eligible role activations for users. If `$roleObject.AssignmentType` equals `null` and the principal is a user, the following KQL query can help out:

```powershell
# KQL query for last PIM role activation
$eligibleRoleLastActivated = $null
$query = "AuditLogs
| where TimeGenerated > ago(90d)
| where OperationName == 'Add member to role completed (PIM activation)'
| where Result == 'success'
| where InitiatedBy.user.id == '$($roleObject.Principal.Id)'
| where TargetResources[0].id == '$($roleObject.RoleDefinition.Id)'
| sort by TimeGenerated desc
| limit 1"

$kqlQuery = Invoke-AzOperationalInsightsQuery -WorkspaceId $WorkspaceID -Query $query
$eligibleRoleLastActivated = $kqlQuery.Results.TimeGenerated
```

### Default MFA method and capability

Users with administrative roles and no registered MFA method can be a security risk, depending on tenant configuration and conditional access policies. It's best to avoid it - while also report on the default type of MFA methods active role assignees have. We already have the `$mfaRegistrationDetailsHashmap` hashtable and can query it for each processed role where the principal is a user.

```powershell
# Fetch default MFA method and cabability
$mfaCapable = $false
$mfaDefaultMethod = $null
if($mfaRegistrationDetailsHashmap.ContainsKey("$($roleObject.Principal.Id)")) {
    $mfaCapable = $mfaRegistrationDetailsHashmap["$($roleObject.Principal.Id)"].IsMfaCapable
    $mfaDefaultMethod = $mfaRegistrationDetailsHashmap["$($roleObject.Principal.Id)"].AdditionalProperties.defaultMfaMethod
}
```

### Admin account owner

If you're following [Microsoft best-practises](https://docs.microsoft.com/en-us/azure/active-directory/roles/security-planning) and separating normal user accounts from administrative roles, you should be having a separate admin account for each user who requires privileged roles and access.

When having separate admin accounts it's also important to check account status of the admin account owners if possible - to make sure that all admin accounts of terminated employees have been disabled and/or deleted. This query will depend on how you identify admin account owners in your tenant, the following example extracts the owner's accountName from the UPN and queries Graph for any user with that `onPremisesSamAccountName` + `employeeId`.

```powershell
# Fetch admin account owner
$adminAccountOwner = $null
if($roleObject.Principal.AdditionalProperties.userPrincipalName -like 'admin-*@<tenant>.onmicrosoft.com') {
    $adminAccountOwnerAccountName = $roleObject.Principal.AdditionalProperties.userPrincipalName -replace "@<tenant>.onmicrosoft.com","" -replace "admin-",""
    $adminAccountOwner = Get-MgUser -Filter "onPremisesSamAccountName eq '$($adminAccountOwnerAccountName)' and employeeId eq '$($roleObject.Principal.AdditionalProperties.employeeId)'" -ConsistencyLevel "eventual" -CountVariable counter -Select "id,userPrincipalName,displayName,onPremisesSamAccountName,employeeId,companyName,department,accountEnabled,signInActivity"
}
```

### Service Principal owner organization

Service Principals of multi-tenant app registrations can be owned by other Azure AD tenants and consented to in your tenant. It's important to know about these and understand why they have privileged roles.

If `$roleObject.Principal.AdditionalProperties.appOwnerOrganizationId` is not `null`, query Graph for the tenant properties of the owner organization.

```powershell
$spOwnerOrg = $null
$spOwnerOrg = Invoke-MgGraphRequest -Method GET -Uri "https://graph.microsoft.com/beta/tenantRelationships/findTenantInformationByTenantId(tenantId='$($roleObject.Principal.AdditionalProperties.appOwnerOrganizationId)')"
```

`$spOwnerOrg.displayName` will contain the tenant organization name, and `$spOwnerOrg.defaultDomainName` the tenant's default domain', which can provide a better clue of what the Service Principal is used for and by whom.

{: .box-warning}
**Note**: Know 100% what you're doing before removing any privileged roles from Service Principals, especially from Microsoft-owned apps which likely have the roles for a very good reason.

That's about it, we now have the data necessary to compile an actionable status report on all active and eligible Azure AD role assignments.

## Compiling the report

We can now construct a `PSCustomObject` per role assignment with the collected data.

```powershell
[PSCustomObject]@{
    'PIM-role last activated' = $eligibleRoleLastActivated
    'Principal Type' = switch ($roleObject.Principal.AdditionalProperties.'@odata.type') {
        '#microsoft.graph.user' { "User" }
        '#microsoft.graph.servicePrincipal' { $roleObject.Principal.AdditionalProperties.servicePrincipalType }
        '#microsoft.graph.group' { "RoleAssignableGroup" }
    }
    'Principal User Type' = $principalSignInActivity.userType
    'Principal Created' = $roleObject.Principal.AdditionalProperties.createdDateTime
    'Principal AD Synced' = $roleObject.Principal.AdditionalProperties.onPremisesSyncEnabled -eq $true
    'Principal Enabled' = $roleObject.Principal.AdditionalProperties.accountEnabled
    'Principal Last SignIn' = $principalLastSignIn
    'Principal DisplayName' = $roleObject.Principal.AdditionalProperties.displayName
    'Principal UPN / AppId' = switch ($roleObject.Principal.AdditionalProperties.'@odata.type') {
        '#microsoft.graph.user' { $roleObject.Principal.AdditionalProperties.userPrincipalName }
        '#microsoft.graph.servicePrincipal' { $roleObject.Principal.AdditionalProperties.appId }
        '#microsoft.graph.group' { "" }
    }
    'Principal Object ID' = $roleObject.Principal.Id
    'Principal Owner' = $spOwnerOrg.displayName + " ($($spOwnerOrg.defaultDomainName))"
    'MFA Capable' = $mfaCapable
    'MFA Default Method' = $mfaDefaultMethod
    'Member Type' = $roleObject.MemberType
    'Assignment Type' = if($roleObject.AssignmentType) { $roleObject.AssignmentType } else { "Eligible" }
    'Directory Scope' = $roleObject.DirectoryScopeId
    'Assigned Role' = $roleObject.roleDefinition.DisplayName
    'Assignment Start Date' = if($roleObject.StartDateTime) { $roleObject.StartDateTime } elseif ($roleObject.scheduleInfo.startDateTime) { $roleObject.scheduleInfo.startDateTime }
    'Assignment End Date' = if($roleObject.EndDateTime) { $roleObject.EndDateTime } elseif ($roleObject.scheduleInfo.expiration.endDateTime) { $roleObject.scheduleInfo.expiration.endDateTime }
    'Has End Date' = $roleObject.EndDateTime -or $roleObject.scheduleInfo.expiration.endDateTime
    'Custom Role' = -not $roleObject.RoleDefinition.IsBuiltIn
    'Role Template' = $roleObject.RoleDefinition.TemplateId
    'AdminOwner Company' = $adminAccountOwner.CompanyName
    'AdminOwner Department' = $adminAccountOwner.Department
    'AdminOwner Name' = $adminAccountOwner.DisplayName
    'AdminOwner UPN' = $adminAccountOwner.UserPrincipalName
    'AdminOwner AccountName' = $adminAccountOwner.OnPremisesSamAccountName
    'AdminOwner EmployeeId' = $adminAccountOwner.EmployeeId
    'AdminOwner Enabled' = $adminAccountOwner.AccountEnabled
    'AdminOwner LastSignIn' = if($adminAccountOwner.SignInActivity) {
        if($adminAccountOwner.SignInActivity.lastSignInDateTime -gt $adminAccountOwner.SignInActivity.lastNonInteractiveSignInDateTime) {
            $adminAccountOwner.SignInActivity.lastSignInDateTime
        } else { $adminAccountOwner.SignInActivity.lastNonInteractiveSignInDateTime }
    }
}
```

### Report examples

User with eligible role assignment:

```text
PIM-role last activated    : 2022-08-24T09:24:20.549Z
Principal Type             : User
Principal User Type        : Member
Principal Created          : 2022-05-11T10:19:28Z
Principal AD Synced        : False
Principal Enabled          : True
Principal Last SignIn      : 18.09.2022 10:49:55
Principal DisplayName      : Adele Vance
Principal UPN / AppId      : AdeleV@tenant.onmicrosoft.com
Principal Object ID        : 806fd75c-2147-40d7-9366-1e3e73d5677b
MFA Capable                : True
MFA Default Method         : microsoftAuthenticatorPush
Member Type                : Direct
Assignment Type            : Eligible
Directory Scope            : /administrativeUnits/bbf65f2a-df92-4e28-8991-555360fa6c98
Assigned Role              : User Administrator
Assignment Start Date      : 23.08.2022 14:07:23
Assignment End Date        : 23.08.2023 14:07:07
Has End Date               : True
Custom Role                : False
Role Template              : fe930be7-5e62-47db-91af-98c3a49a38b1
```

User with active role assignment and owner account details:

```text
Principal Type             : User
Principal User Type        : Member
Principal Created          : 2022-06-14T19:27:21Z
Principal AD Synced        : False
Principal Enabled          : False
Principal Last SignIn      : 17.09.2022 09:37:23
Principal DisplayName      : AdeleV (Admin)
Principal UPN / AppId      : admin-adelev@tenant.onmicrosoft.com
Principal Object ID        : 416e9dc4-9c8d-44ce-8938-1fd9b92334a4
MFA Capable                : True
MFA Default Method         : microsoftAuthenticatorPush
Member Type                : Direct
Assignment Type            : Assigned
Directory Scope            : /
Assigned Role              : Group Creator
Assignment Start Date      : 24.08.2022 09:19:28
Assignment End Date        : 20.02.2023 09:19:13
Has End Date               : True
Custom Role                : True
Role Template              : 8ae1d011-0ae3-4cdf-b6c2-d6fb5cae8254
AdminOwner Company         : Some Company Ltd
AdminOwner Department      : IT
AdminOwner Name            : Adele Vance
AdminOwner UPN             : AdeleV@tenant.onmicrosoft.com
AdminOwner EmployeeId      : 123456
AdminOwner Enabled         : True
AdminOwner LastSignIn      : 18.09.2022 10:49:55
```

Service Principal with role assignment:

```text
Principal Type             : Application
Principal Created          : 2022-05-19T11:57:00Z
Principal AD Synced        : False
Principal Enabled          : True
Principal Last SignIn      : 
Principal DisplayName      : Microsoft.Azure.SyncFabric
Principal UPN / AppId      : 00000014-0000-0000-c000-000000000000
Principal Object ID        : 80faa33c-6cbd-42e5-bb62-4bbd0370351c
Principal Owner            : Microsoft Services (sharepoint.com)
Member Type                : Direct
Assignment Type            : Assigned
Directory Scope            : /
Assigned Role              : Directory Readers
Assignment Start Date      : 
Assignment End Date        : 
Has End Date               : False
Custom Role                : False
Role Template              : 88d8e3e3-8f55-4a1e-953a-9b9898b8876b
```

Managed Identity with role assignment:

```text
Principal Type             : ManagedIdentity
Principal Created          : 2022-06-14T00:03:31Z
Principal AD Synced        : False
Principal Enabled          : True
Principal Last SignIn      : 2022-08-23T11:55:41.73434Z
Principal DisplayName      : test-azrole-grant
Principal UPN / AppId      : 74d64966-c87a-4a4c-a372-fe446b9087ec
Principal Object ID        : e84f76b5-753a-4035-8d1e-c0de1d0686f7
Member Type                : Direct
Assignment Type            : Assigned
Directory Scope            : /
Assigned Role              : Directory Readers
Assignment Start Date      : 19.08.2022 11:09:05
Assignment End Date        : 15.02.2023 11:08:51
Has End Date               : True
Custom Role                : False
Role Template              : 88d8e3e3-8f55-4a1e-953a-9b9898b8876b
```

Role-assignable group with role assignment:

```text
Principal Type             : RoleAssignableGroup
Principal Created          : 2022-08-22T14:08:36Z
Principal AD Synced        : False
Principal DisplayName      : AAD-role PIM: Group Creator
Principal Object ID        : 6a2640b6-df72-4333-a15a-a1dc2056cf77
Member Type                : Direct
Assignment Type            : Assigned
Directory Scope            : /
Assigned Role              : Group Creator
Assignment Start Date      : 
Assignment End Date        : 
Has End Date               : False
Custom Role                : True
Role Template              : 8ae1d011-0ae3-4cdf-b6c2-d6fb5cae8254
```

### Example script

In case you need more tips on creating a reporting powershell script for this report, take a look at the example script I've published on [GitHub](https://github.com/stianstrysse/powershell-scripts/blob/main/AzureAD-AdminRoles-ReportScript.ps1).

Thanks for reading!

Be sure to provide any feedback on [Twitter](https://twitter.com/stianstrysse/status/1571572424516448256) or [LinkedIn](https://www.linkedin.com/posts/stianstrysse_building-a-comprehensive-report-on-azure-activity-6977338056408219649-KTgg).
