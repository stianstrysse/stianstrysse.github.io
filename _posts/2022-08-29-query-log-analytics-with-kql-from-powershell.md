---
layout: post
title: Query Azure AD logs with KQL from Powershell
subtitle: Kusto Query Language (KQL) is a powerfull tool to query Azure AD log entries from Log Anayltics in Azure. See how you can query log data using Powershell.
thumbnail-img: /assets/img/posts/2022-08-29/aad-kql-powershell.png
categories: AZUREAD AZURE KQL LOGANALYTICS POWERSHELL
author: Stian A. Strysse
---

KQL, short for `Kusto Query Language`, is really great for quering data sets like `Sign-in Logs` and `Audit Logs` in Azure AD. KQL is what Microsoft Sentinel uses under the hood for discovering threats, detections and anomalies in larger data sets. But you can also use it to retrieve simpler log entries like:

* Who deleted a specific user.
* How often is a user elevating into an Azure AD administrative role in PIM.
* When was a user added to or removed from a specific Azure AD security group.

Since logs in Azure AD are usually [deleted after 7-30 days](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/reference-reports-data-retention) depending on tenant licensing, it's important to export these logs to a Log Analytics workspace for safekeeping. If the logs aren't exported, there is no way to retrieve them back once they are deleted. You never know when you need to figure out when something happened and who or what actually did it, so having the logs available is key both for security and compliance.

To learn more about KQL I highly recommend [KQL for Microsoft Sentinel](https://github.com/reprise99/Sentinel-Queries#introduction) by Matt Zorich ([@reprise_99](https://twitter.com/reprise_99)), and [Must Learn KQL](https://github.com/rod-trent/MustLearnKQL) by Rod Trent ([@rodtrent](https://twitter.com/rodtrent)).

So, let's get set up for running KQL queries in Powershell.

* [Verify Azure AD diagnostic settings for log export](#verify-azure-ad-diagnostic-settings-for-log-export)
* [Query log analytics from the Azure AD portal](#query-log-analytics-from-the-azure-ad-portal)
* [Query log analytics from Powershell](#query-log-analytics-from-powershell)

## Verify Azure AD diagnostic settings for log export

Check if the Azure AD tenant is already exporting logs by visiting the [Diagnostic settings](https://aad.portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/DiagnosticSettings) blade in the Azure AD portal, any attached Log Analytics workspace will be displayed.

If the Azure AD tenant isn't currently exporting logs to a Log Analytics workspace, see Microsoft's documentation on how to get started:

* [Create a Log Analytics workspace](https://docs.microsoft.com/en-us/azure/azure-monitor/logs/quick-create-workspace?tabs=azure-portal)
* [Integrate Azure AD logs with Azure Monitor logs](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/howto-integrate-activity-logs-with-log-analytics)

## Query log analytics from the Azure AD portal

Go to the [Log Analytics](https://aad.portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/Logs) blade within the Azure AD portal, you will need  `Reader` role on the Log Analytics workspace to query the data.

In the query box, input the following KQL and click `Run`:

```kql
SigninLogs
| where ResultType == 0
| where TimeGenerated > ago(1d)
| where AppDisplayName has "Azure Portal"
```

The results should show all successfull Azure portal sign-ins logged the last 24 hours. Not the most interesting query, but the point is just to show that it works.

![AAD Portal Log Analytics](/assets/img/posts/2022-08-29/aad-portal-log-analytics.png)

Using the `Log Analytics` blade within the Azure AD portal grants quick and easy access to log data, and allows you to play around with queries. But also note that with exported Azure AD log data available in a Log Anayltics workspace, the [built-in workbooks](https://aad.portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/Workbooks) in the Azure AD portal will also light up. These are great for checking things like legacy authentication sign-ins, access package activities, conditional access gaps, provisioning analysis and more. Be sure to give them a run!

## Query log analytics from Powershell

I've started using KQL queries in Powershell automation, especially for auditing and reporting scripts. An example is to discover when a specific user last activated their Azure AD administrative role in PIM - which isn't easily available data without the exported Azure AD logs.

To run KQL queries on Azure AD logs in the Log Analytics workspace, make sure [Azure Powershell module](https://docs.microsoft.com/en-us/powershell/azure/install-az-ps?) is installed. Then it's just a matter of scripting the rest.

Add the correct `subscription`, `log analytics workspace name` and `workspace resource group` to connect with Powershell:

```powershell
# Connect to Azure for Log Analytics
Connect-AzAccount
Set-AzContext -Subscription "my-sub"
$workspaceName = "vl-loganalytics-workspace"
$workspaceRG = "vl-loganalytics"
$WorkspaceID = (Get-AzOperationalInsightsWorkspace -Name $workspaceName -ResourceGroupName $workspaceRG).CustomerID
```

After a successful connection, it's time to run a KQL query. The following query will return the latest Azure AD audit log record for when a specific user objectId last activated their eligible `Global Reader` role in PIM, which is the role definition id `f2ef992c-3afb-46b9-b7cf-a126ee74c451`. All Azure AD administrative roles with IDs are listed on [Microsoft docs](https://docs.microsoft.com/en-us/azure/active-directory/roles/permissions-reference). The `TimeGenerated` property can be set to the number of days you want to check for backwards in time.

```powershell
# Query user's last PIM role activation for 'Global Reader' (role definition id: f2ef992c-3afb-46b9-b7cf-a126ee74c451)
$query = "AuditLogs
| where TimeGenerated > ago(90d)
| where OperationName == 'Add member to role completed (PIM activation)'
| where Result == 'success'
| where InitiatedBy.user.id == '8b51737b-f961-41b0-ade7-6e59c77d6e62'
| where TargetResources[0].id == 'f2ef992c-3afb-46b9-b7cf-a126ee74c451'
| sort by TimeGenerated desc
| limit 1"

$kqlQuery = Invoke-AzOperationalInsightsQuery -WorkspaceId $WorkspaceID -Query $query
$kqlQuery.Results.TimeGenerated
```

The output will either be `null` if a record wasn't found, or the `dateTime` of the user's latest `Global Reader` role activation. `$kqlQuery.Results` will contain the full log entry.

The idea behind this KQL query is to report on users with eligible Azure AD administrative roles never being activated in PIM. Why have an eligible role if it's not being used, right? Anyways, this was just to get you started with KQL for extracting Azure AD log data. Dive into the links I posted at the start of the blogpost to learn more, thanks for reading!

Be sure to provide any feedback on [Twitter](https://twitter.com/stianstrysse/status/1513490814776885249) or [LinkedIn](https://www.linkedin.com/posts/stianstrysse_granting-workload-identities-least-privilege-activity-6919256803994124288-HxZF).
