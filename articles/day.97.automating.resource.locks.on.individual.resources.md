# Day 97 - Automating Resource Locks on Individual Resources in Azure

In [Day 96](./articles.day.96.resource.locks.md), we covered how to implement Resource Locks on individual resources in Azure. Today we are going to show you how you can automate the process of managing Azure Resources that have Resource Locks.

</br>

> **NOTE:** This article was tested and written for a Linux Host running Ubuntu 18.04 with Azure CLI installed.

</br>

In this article:

[Deploy Resources into Azure](#deploy-resources-into-azure) </br>
[Things to Consider](#things-to-consider)</br>
[Conclusion](#conclusion) </br>

## Deploy Resources into Azure

Run the following command to deploy the **100days-reslocks** Resource Group.

```bash
az group create \
--name 100days-reslocks \
--location westeurope
```

You should get back the following output:

```json
{
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/100days-reslocks",
  "location": "westeurope",
  "managedBy": null,
  "name": "100days-reslocks",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null,
  "type": "Microsoft.Resources/resourceGroups"
}
```

</br>

Next, run the following command to create a new VNet in the Resource Group.

```bash
az network vnet create \
--name "100days-reslocks-vnet" \
--resource-group "100days-reslocks" \
--address-prefix "172.16.0.0/16" \
--subnet-name "100days-reslocks-subnet" \
--subnet-prefix "172.16.1.0/24" \
--query "newVNet.provisioningState" \
--output tsv
```

You should get back a similar response.

```console
"Succeeded"
```

Next, run the following command to create a new Azure Key Vault in the Resource Group.

```bash
az keyvault create \
--name "iac100daysreslockskv" \
--resource-group "100days-reslocks" \
--output table
```

You should get back a similar response.

```console
Location    Name                   ResourceGroup
----------  --------------------   ----------------
westeurope  iac100daysreslockskv   100days-reslocks
```

Next, run the following command to create a new Azure Storage Account in the Resource Group.

```bash
/usr/bin/az storage account create \
--name "iac100daysreslocksstr" \
--resource-group "100days-reslocks" \
--sku Standard_LRS \
--query statusOfPrimary \
--output tsv
```

You should get back a similar response.

```console
available
```

</br>

## Lock the Azure Resources

Run the following command to retrieve the **ids** of all of the resources deployed in the **100days-reslocks** Resource Group.

```bash
RESOURCE_IDS=$(az resource list \
--resource-group "100days-reslocks" \
--query [].id \
--output tsv)
```

Run the following command to list the **ids** of the Resources.

```bash
echo "$RESOURCE_IDS"
```

You should get back a similar response.

```console
/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/100days-reslocks/providers/Microsoft.KeyVault/vaults/iac100daysreslockskv
/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/100days-reslocks/providers/Microsoft.Network/virtualNetworks/100days-reslocks-vnet
/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/100days-reslocks/providers/Microsoft.Storage/storageAccounts/iac100daysreslocksstr
```

</br>

Next, run the following command to create a **CanNotDelete** lock for each Resource.

```bash
for ID in $RESOURCE_IDS;
do
    az lock create \
    --name "LockedResources" \
    --notes "100DaysOfIac" \
    --lock-type CanNotDelete \
    --resource $ID \
    --query "id" \
    --output tsv
done
```

You should get back a similar response.

```console
/subscriptions/00000000-0000-0000-0000-000000000000/resourcegroups/100days-reslocks/providers/Microsoft.KeyVault/vaults/iac100daysreslockskv/providers/Microsoft.Authorization/locks/LockedResource
/subscriptions/00000000-0000-0000-0000-000000000000/resourcegroups/100days-reslocks/providers/Microsoft.Network/virtualNetworks/100days-reslocks-vnet/providers/Microsoft.Authorization/locks/LockedResource
/subscriptions/00000000-0000-0000-0000-000000000000/resourcegroups/100days-reslocks/providers/Microsoft.Storage/storageAccounts/iac100daysreslocksstr/providers/Microsoft.Authorization/locks/LockedResource
```

</br>

> **NOTE:** You will receive an error if you attempt to delete the Resource Group while the resource locks are in place.

</br>

## Tag the Resources

Tagging your Azure Resources is probably one of the easiest ways to not only keep track of them in Azure, but also to keep track what type of state they should be in. This could be anything from always being on to retired to permanent removal. The range of states you want to track your resources by is entirely up to you.

Next, we are going to Tag all of the Resources in the Resource Group.

```bash
for ID in $RESOURCE_IDS;
do
    az resource tag \
    --tags "Permanent=True" \
    --ids $ID \
    --query 'id' \
    --output tsv
done
```

You should get back a similar response to what is shown below.

```console
/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/100days-reslocks/providers/Microsoft.KeyVault/vaults/iac100daysreslockskv
/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/100days-reslocks/providers/Microsoft.Network/virtualNetworks/100days-reslocks-vnet
/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/100days-reslocks/providers/Microsoft.Storage/storageAccounts/iac100daysreslocksstr
```

</br>

## Update the Tag on the Storage Account

Next, we are going to change the Storage Account's **Permanent** Tag from *True* to *False*.

Run the following command to retrieve the **id** of the Storage Account.

```bash
STORAGE_ID=$(az resource list \
--resource-group "100days-reslocks" \
| jq '.[].id | select(.|test("iac100daysreslocksstr"))' | tr -d '"')
```

</br>

Next, run the following command to change the Storage Account's **Permanent** Tag from *True* to *False*.

```bash
az resource tag \
--tags "Permanent=False" \
--ids $STORAGE_ID \
--query 'id' \
--output tsv
```

You should get back a similar response to what is shown below.

```console
/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/100days-reslocks/providers/Microsoft.Storage/storageAccounts/iac100daysreslocksstr
```

</br>

## Delete a Locked Resource Based on a Tag Value

Finally, we are going to delete the Storage Account from the Resource Group based on it's **Permanent** Tag being changed from *True* to *False*.

<br/>

Run the following command below to delete the Storage Account in the Resource Group. If you prefer, you can also paste the content below into a **.sh** file, make the file executable, and then run it.

```bash
for ID in $RESOURCE_IDS;
do
    # Retrieving the Resource 'Permanent' Tag.
    CHECK_TAG=$(az resource show \
    --ids $ID \
    --query "tags.Permanent" \
    --output tsv)

    if [[ "$CHECK_TAG" == "True" ]]; then
        echo "[---info------] Not Marked for Removal, skipping."
    else
        echo "[---info------] [$ID] is marked for removal."

        # Removing the Lock on the Resource.
        REMOVE_LOCK=$(az lock delete \
        --name "LockedResources" \
        --resource $ID)

        if [ $? -eq 0 ]; then
            echo "[---success---] Removed Resource Lock on [$ID]."
        else
            echo "[---fail------] Failed to remove Resource Lock on [$ID]."
            echo "[---fail------] $REMOVE_LOCK"
        fi

        # Deleting the Resource.
        DELETE_RESOURCE=$(az resource delete --ids $ID)

        if [ $? -eq 0 ]; then
            echo "[---success---] Deleted Resource [$ID]."
        else
            echo "[---fail------] Failed to delete Resource[$ID]."
            echo "[---fail------] $DELETE_RESOURCE"
        fi
    fi
done
```

You should get back a similar response to what is shown below.

```console

```

</br>

## Things to Consider

The methodology shown above should work for just about all Resources in Azure that are taggable. If you plan on implementing this type of solution, we recommend that you test this against your Azure Resources thoroughly before doing so.

</br>

## Conclusion

In today's article we demonstrated how you can automate the management of Resource Locks on Individual Resources in Azure. In the next article, we'll be demonstrating how you can use this for unlocking and relocking resources that are updated from a YAML Pipeline. If there's a specific scenario that you wish to be covered in future articles, please create a **[New Issue](https://github.com/starkfell/100DaysOfIaC/issues)** in the [starkfell/100DaysOfIaC](https://github.com/starkfell/100DaysOfIaC/) GitHub repository.