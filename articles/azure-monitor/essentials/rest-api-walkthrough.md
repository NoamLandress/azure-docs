---
title: Azure Monitoring REST API walkthrough
description: How to authenticate requests and use the Azure Monitor REST API to retrieve available metric definitions and metric values.
ms.topic: conceptual
ms.date: 05/09/2022
ms.custom: has-adal-ref, devx-track-azurepowershell
ms.reviewer: robb
---

# Azure Monitoring REST API walkthrough


This article shows you how to perform authentication so your code can use the [Microsoft Azure Monitor REST API Reference](/rest/api/monitor/).

The Azure Monitor API makes it possible to programmatically retrieve the available default metric definitions, dimension values, and metric values. The data can be saved in a separate data store such as Azure SQL Database, Azure Cosmos DB, or Azure Data Lake. From there additional analysis can be performed as needed.

Besides working with various metric data points, the Monitor API also makes it possible to list alert rules, view activity logs, and much more. For a full list of available operations, see the [Microsoft Azure Monitor REST API Reference](/rest/api/monitor/).  

## Authenticating Azure Monitor requests  
   

All the tasks executed against the Azure Monitor API use the Azure Resource Manager authentication model. Therefore, all requests must be authenticated with Azure Active Directory (Azure AD). One approach to authenticate the client application is to create an Azure AD service principal and retrieve the authentication (JWT) token. The following sample script demonstrates creating an Azure AD service principal via PowerShell. For a more detailed walk-through, refer to the documentation on [using Azure PowerShell to create a service principal to access resources](/powershell/azure/create-azure-service-principal-azureps). It is also possible to [create a service principal via the Azure portal](../../active-directory/develop/howto-create-service-principal-portal.md).

```powershell
$subscriptionId = "{azure-subscription-id}"
$resourceGroupName = "{resource-group-name}"

# Authenticate to a specific Azure subscription.
Connect-AzAccount -SubscriptionId $subscriptionId

# Password for the service principal
$pwd = "{service-principal-password}"
$secureStringPassword = ConvertTo-SecureString -String $pwd -AsPlainText -Force

# Create a new Azure AD application
$azureAdApplication = New-AzADApplication `
                        -DisplayName "My Azure Monitor" `
                        -HomePage "https://localhost/azure-monitor" `
                        -IdentifierUris "https://localhost/azure-monitor" `
                        -Password $secureStringPassword

# Create a new service principal associated with the designated application
New-AzADServicePrincipal -ApplicationId $azureAdApplication.ApplicationId

# Assign Reader role to the newly created service principal
New-AzRoleAssignment -RoleDefinitionName Reader `
                          -ServicePrincipalName $azureAdApplication.ApplicationId.Guid

```

To query the Azure Monitor API, the client application should use the previously created service principal to authenticate. The following example PowerShell script shows one approach, using the [Microsoft Authentication Library (MSAL)](../../active-directory/develop/msal-overview.md) to obtain the authentication token.

```powershell
$ClientID           = "{client_id}"
$loginURL           = "https://login.microsoftonline.com"
$tenantdomain       = "{tenant_id}"
$CertPassWord       = "{password_for_cert}"
$certPath           = "C:\temp\Certs\testCert_01.pfx"
 
[string[]] $Scopes  = "https://graph.microsoft.com/.default"
 
Function Load-MSAL {
    if ($PSVersionTable.PSVersion.Major -gt 5)
    { 
        $core = $true
        $foldername =  "netcoreapp2.1"
    }
    else
    { 
        $core = $false
        $foldername = "net45"
    }
 
    # Download MSAL.Net module to a local folder if it does not exist there
    if ( ! (Get-ChildItem $HOME/MSAL/lib/Microsoft.Identity.Client.* -erroraction ignore) ) {
        install-package -Source nuget.org -ProviderName nuget -SkipDependencies Microsoft.Identity.Client -Destination $HOME/MSAL/lib -force -forcebootstrap | out-null
    }
   
    # Load the MSAL assembly -- needed once per PowerShell session
    [System.Reflection.Assembly]::LoadFrom((Get-ChildItem $HOME/MSAL/lib/Microsoft.Identity.Client.*/lib/$foldername/Microsoft.Identity.Client.dll).fullname) | out-null
  }
  
Function Get-GraphAccessTokenFromMSAL {
 
    Load-MSAL
 
    $global:app = $null
 
    $x509cert = [System.Security.Cryptography.X509Certificates.X509Certificate2] (GetX509Certificate_FromPfx -CertificatePath $certPath -CertificatePassword $CertPassWord)
    write-host "Cert = {$x509cert}"
 
    $ClientApplicationBuilder = [Microsoft.Identity.Client.ConfidentialClientApplicationBuilder]::Create($ClientID)
        [void]$ClientApplicationBuilder.WithAuthority($("$loginURL/$tenantdomain"))
        [void]$ClientApplicationBuilder.WithCertificate($x509cert)
    $global:app = $ClientApplicationBuilder.Build()
 
    [Microsoft.Identity.Client.AuthenticationResult] $authResult  = $null
    $AquireTokenParameters = $global:app.AcquireTokenForClient($Scopes)
    try {
        $authResult = $AquireTokenParameters.ExecuteAsync().GetAwaiter().GetResult()
    }
    catch {
        $ErrorMessage = $_.Exception.Message
        Write-Host $ErrorMessage
    }
     
    return $authResult
}
 
function GetX509Certificate_FromPfx($CertificatePath, $CertificatePassword){
    #write-host "Path: '$CertificatePath'"
    
    if(![System.IO.Path]::IsPathRooted($CertificatePath))
    {
        $LocalPath = Get-Location
        $CertificatePath = "$LocalPath\$CertificatePath"
    }
 
    #Write-Host "Looking for '$CertificatePath'"
 
    $certificate = [System.Security.Cryptography.X509Certificates.X509Certificate2]::new($CertificatePath, $CertificatePassword)
     
    Return $certificate
}
 
$myvar = Get-GraphAccessTokenFromMSAL
Write-Host "Access Token: " $myvar.AccessToken
 
```
Loading the certificate from a .pfx file in PowerShell can make it easier for an admin to manage certificates without having to install the certificate in the certificate store. However, this should not be done on a client machine as the user could potentially discover the file and also the password for it, as well as the method to authenticate. The client credentials flow is only intended to be ran in a back-end service to service type of scenario where only admins have access to the machine.

After authenticating, queries can then be executed against the Azure Monitor REST API. There are two helpful queries:

1. List the metric definitions for a resource
2. Retrieve the metric values

> [!NOTE]
> For additional information on authenticating with the Azure REST API, please refer to the [Azure REST API Reference](/rest/api/azure/).
>
>

## Retrieve Metric Definitions

Use the [Azure Monitor Metric definitions REST API](/rest/api/monitor/metricdefinitions) to access the list of metrics that are available for a service.

**Method**: GET

**Request URI**: https:\/\/management.azure.com/subscriptions/*{subscriptionId}*/resourceGroups/*{resourceGroupName}*/providers/*{resourceProviderNamespace}*/*{resourceType}*/*{resourceName}*/providers/microsoft.insights/metricDefinitions?api-version=*{apiVersion}*

For example, to retrieve the metric definitions for an Azure Storage account, the request would appear as follows:

```powershell
$request = "https://management.azure.com/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/azmon-rest-api-walkthrough/providers/Microsoft.Storage/storageAccounts/ContosoStorage/providers/microsoft.insights/metricDefinitions?api-version=2018-01-01"


Invoke-RestMethod -Uri $request ` 
                  -Headers $authHeader `
                  -Method Get `
                  -OutFile ".\contosostorage-metricdef-results.json" `
                  -Verbose

```

> [!NOTE]
> Older versions of the metric definitions API did not support dimensions. We recommend using API version "2018-01-01" or later.
>
>

The resulting JSON response body would be similar to the following example: (Note that the second metric has dimensions)

```json
{
    "value": [
        {
            "id": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/azmon-rest-api-walkthrough/providers/Microsoft.Storage/storageAccounts/ContosoStorage/providers/microsoft.insights/metricdefinitions/UsedCapacity",
            "resourceId": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/azmon-rest-api-walkthrough/providers/Microsoft.Storage/storageAccounts/ContosoStorage",
            "namespace": "Microsoft.Storage/storageAccounts",
            "category": "Capacity",
            "name": {
                "value": "UsedCapacity",
                "localizedValue": "Used capacity"
            },
            "isDimensionRequired": false,
            "unit": "Bytes",
            "primaryAggregationType": "Average",
            "supportedAggregationTypes": [
                "Total",
                "Average",
                "Minimum",
                "Maximum"
            ],
            "metricAvailabilities": [
                {
                    "timeGrain": "PT1H",
                    "retention": "P93D"
                },
                {
                    "timeGrain": "PT6H",
                    "retention": "P93D"
                },
                {
                    "timeGrain": "PT12H",
                    "retention": "P93D"
                },
                {
                    "timeGrain": "P1D",
                    "retention": "P93D"
                }
            ]
        },
        {
            "id": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/azmon-rest-api-walkthrough/providers/Microsoft.Storage/storageAccounts/ContosoStorage/providers/microsoft.insights/metricdefinitions/Transactions",
            "resourceId": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/azmon-rest-api-walkthrough/providers/Microsoft.Storage/storageAccounts/ContosoStorage",
            "namespace": "Microsoft.Storage/storageAccounts",
            "category": "Transaction",
            "name": {
                "value": "Transactions",
                "localizedValue": "Transactions"
            },
            "isDimensionRequired": false,
            "unit": "Count",
            "primaryAggregationType": "Total",
            "supportedAggregationTypes": [
                "Total"
            ],
            "metricAvailabilities": [
                {
                    "timeGrain": "PT1M",
                    "retention": "P93D"
                },
                {
                    "timeGrain": "PT5M",
                    "retention": "P93D"
                },
                {
                    "timeGrain": "PT15M",
                    "retention": "P93D"
                },
                {
                    "timeGrain": "PT30M",
                    "retention": "P93D"
                },
                {
                    "timeGrain": "PT1H",
                    "retention": "P93D"
                },
                {
                    "timeGrain": "PT6H",
                    "retention": "P93D"
                },
                {
                    "timeGrain": "PT12H",
                    "retention": "P93D"
                },
                {
                    "timeGrain": "P1D",
                    "retention": "P93D"
                }
            ],
            "dimensions": [
                {
                    "value": "ResponseType",
                    "localizedValue": "Response type"
                },
                {
                    "value": "GeoType",
                    "localizedValue": "Geo type"
                },
                {
                    "value": "ApiName",
                    "localizedValue": "API name"
                }
            ]
        },
        ...
    ]
}
```

## Retrieve Dimension Values

Once the available metric definitions are known, there may be some metrics that have dimensions. Before querying for the metric you may want to discover what the range of values a dimension has. Based on these dimension values you can then choose to filter or segment the metrics based on dimension values while querying for metrics.  Use the [Azure Monitor Metrics REST API](/rest/api/monitor/metrics) to find all the possible values for a given metric dimension.

Use the metric’s name ‘value’ (not the ‘localizedValue’) for any filtering requests. If no filters are specified, the default metric is returned. The usage of this API only allows one dimension to have a wildcard filter. The key difference between a dimension values request and a metric data request is specifying the "resultType=metadata" query parameter.

> [!NOTE]
> To retrieve dimension values using the Azure Monitor REST API, use "2019-07-01" the API version or later.
>
>

**Method**: GET

**Request URI**: https\://management.azure.com/subscriptions/*{subscription-id}*/resourceGroups/*{resource-group-name}*/providers/*{resource-provider-namespace}*/*{resource-type}*/*{resource-name}*/providers/microsoft.insights/metrics?metricnames=*{metric}*&timespan=*{starttime/endtime}*&$filter=*{filter}*&resultType=metadata&api-version=*{apiVersion}*

For example, to retrieve the list of dimension values that were emitted for the 'API Name' dimension for the 'Transactions' metric, where the GeoType dimension = 'Primary'  during the specified time range, the request would be as follows:

```powershell
$filter = "APIName eq '*' and GeoType eq 'Primary'"
$request = "https://management.azure.com/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/azmon-rest-api-walkthrough/providers/Microsoft.Storage/storageAccounts/ContosoStorage/providers/microsoft.insights/metrics?metricnames=Transactions&timespan=2018-03-01T00:00:00Z/2018-03-02T00:00:00Z&resultType=metadata&`$filter=GeoType eq 'Primary' and ApiName eq '*'&api-version=2019-07-01"
Invoke-RestMethod -Uri $request `
    -Headers $authHeader `
    -Method Get `
    -OutFile ".\contosostorage-dimension-values.json" `
    -Verbose
```

The resulting JSON response body would be similar to the following example:

```json
{
  "timespan": "2018-03-01T00:00:00Z/2018-03-02T00:00:00Z",
  "value": [
    {
      "id": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/azmon-rest-api-walkthrough/providers/Microsoft.Storage/storageAccounts/ContosoStorage/providers/Microsoft.Insights/metrics/Transactions",
      "type": "Microsoft.Insights/metrics",
      "name": {
        "value": "Transactions",
        "localizedValue": "Transactions"
      },
      "unit": "Count",
      "timeseries": [
        {
          "metadatavalues": [
            {
              "name": {
                "value": "apiname",
                "localizedValue": "apiname"
              },
              "value": "DeleteBlob"
            }
          ]
        },
        {
          "metadatavalues": [
            {
              "name": {
                "value": "apiname",
                "localizedValue": "apiname"
              },
              "value": "SetBlobProperties"
            }
          ]
        },
        ...
      ]
    }
  ],
  "namespace": "Microsoft.Storage/storageAccounts",
  "resourceregion": "eastus"
}
```

## Retrieve Metric Values

Once the available metric definitions and possible dimension values are known, it is then possible to retrieve the related metric values.  Use the [Azure Monitor Metrics REST API](/rest/api/monitor/metrics) to achieve this.

Use the metric’s name ‘value’ (not the ‘localizedValue’) for any filtering requests. If no dimension filters are specified, the rolled up aggregated metric is returned. To fetch multiple time series with specific dimension values, specify a filter query parameter that specifies both dimension values such as "&$filter=ApiName eq 'ListContainers' or ApiName eq 'GetBlobServiceProperties'". To return a time series for every value of a given dimension, use a '*' filter such as "&$filter=ApiName eq '*'". The 'Top' and 'OrderBy' query parameters can be used to limit and order the number of time series returned.

> [!NOTE]
> To retrieve multi-dimensional metric values using the Azure Monitor REST API, use "2019-07-01" the API version or later.
>
>

**Method**: GET

**Request URI**: https:\//management.azure.com/subscriptions/*{subscription-id}*/resourceGroups/*{resource-group-name}*/providers/*{resource-provider-namespace}*/*{resource-type}*/*{resource-name}*/providers/microsoft.insights/metrics?metricnames=*{metric}*&timespan=*{starttime/endtime}*&$filter=*{filter}*&interval=*{timeGrain}*&aggregation=*{aggreation}*&api-version=*{apiVersion}*

For example, to retrieve the top 3 APIs, in descending value, by the number of 'Transactions' during a 5 min range, where the GeotType was 'Primary', the request would be as follows:

```powershell
$filter = "APIName eq '*' and GeoType eq 'Primary'"
$request = "https://management.azure.com/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/azmon-rest-api-walkthrough/providers/Microsoft.Storage/storageAccounts/ContosoStorage/providers/microsoft.insights/metrics?metricnames=Transactions&timespan=2018-03-01T02:00:00Z/2018-03-01T02:05:00Z&`$filter=apiname eq 'GetBlobProperties'&interval=PT1M&aggregation=Total&top=3&orderby=Total desc&api-version=2019-07-01"
Invoke-RestMethod -Uri $request `
    -Headers $authHeader `
    -Method Get `
    -OutFile ".\contosostorage-metric-values.json" `
    -Verbose
```

The resulting JSON response body would be similar to the following example:

```json
{
  "cost": 0,
  "timespan": "2018-03-01T02:00:00Z/2018-03-01T02:05:00Z",
  "interval": "PT1M",
  "value": [
    {
      "id": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/azmon-rest-api-walkthrough/providers/Microsoft.Storage/storageAccounts/ContosoStorage/providers/Microsoft.Insights/metrics/Transactions",
      "type": "Microsoft.Insights/metrics",
      "name": {
        "value": "Transactions",
        "localizedValue": "Transactions"
      },
      "unit": "Count",
      "timeseries": [
        {
          "metadatavalues": [
            {
              "name": {
                "value": "apiname",
                "localizedValue": "apiname"
              },
              "value": "GetBlobProperties"
            }
          ],
          "data": [
            {
              "timeStamp": "2017-09-19T02:00:00Z",
              "total": 2
            },
            {
              "timeStamp": "2017-09-19T02:01:00Z",
              "total": 1
            },
            {
              "timeStamp": "2017-09-19T02:02:00Z",
              "total": 3
            },
            {
              "timeStamp": "2017-09-19T02:03:00Z",
              "total": 7
            },
            {
              "timeStamp": "2017-09-19T02:04:00Z",
              "total": 2
            }
          ]
        },
        ...
      ]
    }
  ],
  "namespace": "Microsoft.Storage/storageAccounts",
  "resourceregion": "eastus"
}
```

### Use ARMClient

Another approach is to use [ARMClient](https://github.com/projectkudu/armclient) on your Windows machine. ARMClient handles the Azure AD authentication (and resulting JWT token) automatically. The following steps outline the use of ARMClient for retrieving metric data:

1. Install [Chocolatey](https://chocolatey.org/) and [ARMClient](https://github.com/projectkudu/armclient).
2. In a terminal window, type *armclient.exe login*. Doing so prompts you to log in to Azure.
3. Type *armclient GET [your_resource_id]/providers/microsoft.insights/metricdefinitions?api-version=2016-03-01*
4. Type *armclient GET [your_resource_id]/providers/microsoft.insights/metrics?api-version=2016-09-01*

For example, in order to retrieve the metric definitions for a specific Logic App, issue the following command:

```console
armclient GET /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/azmon-rest-api-walkthrough/providers/Microsoft.Logic/workflows/ContosoTweets/providers/microsoft.insights/metricDefinitions?api-version=2016-03-01
```

## Retrieve the resource ID

Using the REST API can really help to understand the available metric definitions, granularity, and related values. That information is helpful when using the [Azure Management Library](/previous-versions/azure/reference/mt417623(v=azure.100)).

For the preceding code, the resource ID to use is the full path to the desired Azure resource. For example, to query against an Azure Web App, the resource ID would be:

*/subscriptions/{subscription-id}/resourceGroups/{resource-group-name}/providers/Microsoft.Web/sites/{site-name}/*

The following list contains a few examples of resource ID formats for various Azure resources:

* **IoT Hub** - /subscriptions/*{subscription-id}*/resourceGroups/*{resource-group-name}*/providers/Microsoft.Devices/IotHubs/*{iot-hub-name}*
* **Elastic SQL Pool** - /subscriptions/*{subscription-id}*/resourceGroups/*{resource-group-name}*/providers/Microsoft.Sql/servers/*{pool-db}*/elasticpools/*{sql-pool-name}*
* **SQL Database (v12)** - /subscriptions/*{subscription-id}*/resourceGroups/*{resource-group-name}*/providers/Microsoft.Sql/servers/*{server-name}*/databases/*{database-name}*
* **Service Bus** - /subscriptions/*{subscription-id}*/resourceGroups/*{resource-group-name}*/providers/Microsoft.ServiceBus/*{namespace}*/*{servicebus-name}*
* **Virtual machine scale sets** - /subscriptions/*{subscription-id}*/resourceGroups/*{resource-group-name}*/providers/Microsoft.Compute/virtualMachineScaleSets/*{vm-name}*
* **VMs** - /subscriptions/*{subscription-id}*/resourceGroups/*{resource-group-name}*/providers/Microsoft.Compute/virtualMachines/*{vm-name}*
* **Event Hubs** - /subscriptions/*{subscription-id}*/resourceGroups/*{resource-group-name}*/providers/Microsoft.EventHub/namespaces/*{eventhub-namespace}*

There are alternative approaches to retrieving the resource ID, including using Azure Resource Explorer, viewing the desired resource in the Azure portal, and via PowerShell or the Azure CLI.

### Azure Resource Explorer

To find the resource ID for a desired resource, one helpful approach is to use the [Azure Resource Explorer](https://resources.azure.com) tool. Navigate to the desired resource and then look at the ID shown, as in the following screenshot:

![Alt "Azure Resource Explorer"](./media/rest-api-walkthrough/azure_resource_explorer.png)

### Azure portal

The resource ID can also be obtained from the Azure portal. To do so, navigate to the desired resource and then select Properties. The Resource ID is displayed in the Properties section, as seen in the following screenshot:

![Alt "Resource ID displayed in the Properties blade in the Azure portal"](./media/rest-api-walkthrough/resourceid_azure_portal.png)

### Azure PowerShell

The resource ID can be retrieved using Azure PowerShell cmdlets as well. For example, to obtain the resource ID for an Azure Logic App, execute the Get-AzureLogicApp cmdlet, as in the following example:

```powershell
Get-AzLogicApp -ResourceGroupName azmon-rest-api-walkthrough -Name contosotweets
```

The result should be similar to the following example:

```output
Id             : /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/azmon-rest-api-walkthrough/providers/Microsoft.Logic/workflows/ContosoTweets
Name           : ContosoTweets
Type           : Microsoft.Logic/workflows
Location       : centralus
ChangedTime    : 8/21/2017 6:58:57 PM
CreatedTime    : 8/18/2017 7:54:21 PM
AccessEndpoint : https://prod-08.centralus.logic.azure.com:443/workflows/f3a91b352fcc47e6bff989b85446c5db
State          : Enabled
Definition     : {$schema, contentVersion, parameters, triggers...}
Parameters     : {[$connections, Microsoft.Azure.Management.Logic.Models.WorkflowParameter]}
SkuName        :
AppServicePlan :
PlanType       :
PlanId         :
Version        : 08586982649483762729
```

### Azure CLI

To retrieve the resource ID for an Azure Storage account using the Azure CLI, execute the `az storage account show` command, as shown in the following example:

```azurecli
az storage account show -g azmon-rest-api-walkthrough -n contosotweets2017
```

The result should be similar to the following example:

```json
{
  "accessTier": null,
  "creationTime": "2017-08-18T19:58:41.840552+00:00",
  "customDomain": null,
  "enableHttpsTrafficOnly": false,
  "encryption": null,
  "id": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/azmon-rest-api-walkthrough/providers/Microsoft.Storage/storageAccounts/contosotweets2017",
  "identity": null,
  "kind": "Storage",
  "lastGeoFailoverTime": null,
  "location": "centralus",
  "name": "contosotweets2017",
  "networkAcls": null,
  "primaryEndpoints": {
    "blob": "https://contosotweets2017.blob.core.windows.net/",
    "file": "https://contosotweets2017.file.core.windows.net/",
    "queue": "https://contosotweets2017.queue.core.windows.net/",
    "table": "https://contosotweets2017.table.core.windows.net/"
  },
  "primaryLocation": "centralus",
  "provisioningState": "Succeeded",
  "resourceGroup": "azmon-rest-api-walkthrough",
  "secondaryEndpoints": null,
  "secondaryLocation": "eastus2",
  "sku": {
    "name": "Standard_GRS",
    "tier": "Standard"
  },
  "statusOfPrimary": "available",
  "statusOfSecondary": "available",
  "tags": {},
  "type": "Microsoft.Storage/storageAccounts"
}
```

> [!NOTE]
> Azure Logic Apps are not yet available via the Azure CLI, thus an Azure Storage account is shown in the preceding example.
>
>

## Retrieve activity log data

In addition to metric definitions and related values, it is also possible to use the Azure Monitor REST API to retrieve other interesting insights related to Azure resources. As an example, it is possible to query [activity log](/rest/api/monitor/activitylogs) data. The following sample requests use the Azure Monitor REST API to query the activity log.

Get Activity Logs with filter:

``` HTTP
GET https://management.azure.com/subscriptions/089bd33f-d4ec-47fe-8ba5-0753aa5c5b33/providers/microsoft.insights/eventtypes/management/values?api-version=2015-04-01&$filter=eventTimestamp ge '2018-01-21T20:00:00Z' and eventTimestamp le '2018-01-23T20:00:00Z' and resourceGroupName eq 'MSSupportGroup'
```

Get Activity Logs with filter and select:

```HTTP
GET https://management.azure.com/subscriptions/089bd33f-d4ec-47fe-8ba5-0753aa5c5b33/providers/microsoft.insights/eventtypes/management/values?api-version=2015-04-01&$filter=eventTimestamp ge '2015-01-21T20:00:00Z' and eventTimestamp le '2015-01-23T20:00:00Z' and resourceGroupName eq 'MSSupportGroup'&$select=eventName,id,resourceGroupName,resourceProviderName,operationName,status,eventTimestamp,correlationId,submissionTimestamp,level
```

Get Activity Logs with select:

```HTTP
GET https://management.azure.com/subscriptions/089bd33f-d4ec-47fe-8ba5-0753aa5c5b33/providers/microsoft.insights/eventtypes/management/values?api-version=2015-04-01&$select=eventName,id,resourceGroupName,resourceProviderName,operationName,status,eventTimestamp,correlationId,submissionTimestamp,level
```

Get Activity Logs without filter or select:

```HTTP
GET https://management.azure.com/subscriptions/089bd33f-d4ec-47fe-8ba5-0753aa5c5b33/providers/microsoft.insights/eventtypes/management/values?api-version=2015-04-01
```

## Troubleshooting

If you receive a 429, 503, or 504 error, please retry the API in one minute.


## Next steps

* Review the [Overview of Monitoring](../overview.md).
* View the [Supported metrics with Azure Monitor](./metrics-supported.md).
* Review the [Microsoft Azure Monitor REST API Reference](/rest/api/monitor/).
* Review the [Azure Management Library](/previous-versions/azure/reference/mt417623(v=azure.100)).