---
title: Create resource lock for a Gremlin database and graph for Azure Cosmos DB
description: Create resource lock for a Gremlin database and graph for Azure Cosmos DB
author: seesharprun
ms.author: sidandrews
ms.reviewer: mjbrown
ms.service: cosmos-db
ms.subservice: cosmosdb-graph
ms.topic: sample
ms.date: 02/21/2022
---

# Create a resource lock for Azure Cosmos Gremlin API database and graph using Azure CLI

[!INCLUDE[appliesto-gremlin-api](../../../includes/appliesto-gremlin-api.md)]

The script in this article demonstrates performing resource lock operations for a Gremlin database and graph.

> [!IMPORTANT]
>
> To create resource locks, you must have membership in the owner role in the subscription.
>
> Resource locks do not work for changes made by users connecting using any Gremlin SDK or the Azure Portal unless the Cosmos DB account is first locked with the `disableKeyBasedMetadataWriteAccess` property enabled. To learn more about how to enable this property see, [Preventing changes from SDKs](../../../role-based-access-control.md#prevent-sdk-changes).

[!INCLUDE [quickstarts-free-trial-note](../../../../../includes/quickstarts-free-trial-note.md)]

[!INCLUDE [azure-cli-prepare-your-environment.md](../../../../../includes/azure-cli-prepare-your-environment.md)]

- This article requires version 2.30 or later. Run `az --version` to find the version. If you need to install or upgrade, see [Install Azure CLI](/cli/azure/install-azure-cli). If using Azure Cloud Shell, the latest version is already installed.

## Sample script

[!INCLUDE [cli-launch-cloud-shell-sign-in.md](../../../../../includes/cli-launch-cloud-shell-sign-in.md)]

### Run the script

:::code language="azurecli" source="~/azure_cli_scripts/cosmosdb/gremlin/lock.sh" id="FullScript":::

## Clean up resources

[!INCLUDE [cli-clean-up-resources.md](../../../../../includes/cli-clean-up-resources.md)]

```azurecli
az group delete --name $resourceGroup
```

## Sample reference

This script uses the following commands. Each command in the table links to command specific documentation.

| Command | Notes |
|---|---|
| [az lock create](/cli/azure/lock#az-lock-create) | Creates a lock. |
| [az lock list](/cli/azure/lock#az-lock-list) | List lock information. |
| [az lock show](/cli/azure/lock#az-lock-show) | Show properties of a lock. |
| [az lock delete](/cli/azure/lock#az-lock-delete) | Deletes a lock. |

## Next steps

- [Lock resources to prevent unexpected changes](../../../../azure-resource-manager/management/lock-resources.md)

- [Azure Cosmos DB CLI documentation](/cli/azure/cosmosdb).

- [Azure Cosmos DB CLI GitHub Repository](https://github.com/Azure-Samples/azure-cli-samples/tree/master/cosmosdb).
