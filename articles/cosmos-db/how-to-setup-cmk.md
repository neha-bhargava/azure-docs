---
title: Configure customer-managed keys for your Azure Cosmos DB account
description: Learn how to configure customer-managed keys for your Azure Cosmos DB account with Azure Key Vault
author: ThomasWeiss
ms.service: cosmos-db
ms.topic: how-to
ms.date: 02/03/2022
ms.author: thweiss 
ms.custom: devx-track-azurepowershell
---

# Configure customer-managed keys for your Azure Cosmos account with Azure Key Vault
[!INCLUDE[appliesto-all-apis](includes/appliesto-all-apis.md)]

Data stored in your Azure Cosmos account is automatically and seamlessly encrypted with keys managed by Microsoft (**service-managed keys**). Optionally, you can choose to add a second layer of encryption with keys you manage (**customer-managed keys** or CMK).

:::image type="content" source="./media/how-to-setup-cmk/cmk-intro.png" alt-text="Layers of encryption around customer data":::

You must store customer-managed keys in [Azure Key Vault](../key-vault/general/overview.md) and provide a key for each Azure Cosmos account that is enabled with customer-managed keys. This key is used to encrypt all the data stored in that account.

> [!NOTE]
> Currently, customer-managed keys are available only for new Azure Cosmos accounts. You should configure them during account creation.

## <a id="register-resource-provider"></a> Register the Azure Cosmos DB resource provider for your Azure subscription

1. Sign in to the [Azure portal](https://portal.azure.com/), go to your Azure subscription, and select **Resource providers** under the **Settings** tab:

   :::image type="content" source="./media/how-to-setup-cmk/portal-rp.png" alt-text="Resource providers entry from the left menu":::

1. Search for the **Microsoft.DocumentDB** resource provider. Verify if the resource provider is already marked as registered. If not, choose the resource provider and select **Register**:

   :::image type="content" source="./media/how-to-setup-cmk/portal-rp-register.png" alt-text="Registering the Microsoft.DocumentDB resource provider":::

## Configure your Azure Key Vault instance

> [!IMPORTANT]
> Your Azure Key Vault instance must be accessible through public network access. An instance that is only accessible through [private endpoints](../key-vault/general/private-link-service.md) cannot be used to host your customer-managed keys.

Using customer-managed keys with Azure Cosmos DB requires you to set two properties on the Azure Key Vault instance that you plan to use to host your encryption keys: **Soft Delete** and **Purge Protection**.

If you create a new Azure Key Vault instance, enable these properties during creation:

:::image type="content" source="./media/how-to-setup-cmk/portal-akv-prop.png" alt-text="Enabling soft delete and purge protection for a new Azure Key Vault instance":::

If you're using an existing Azure Key Vault instance, you can verify that these properties are enabled by looking at the **Properties** section on the Azure portal. If any of these properties isn't enabled, see the "Enabling soft-delete" and "Enabling Purge Protection" sections in one of the following articles:

- [How to use soft-delete with PowerShell](../key-vault/general/key-vault-recovery.md)
- [How to use soft-delete with Azure CLI](../key-vault/general/key-vault-recovery.md)

## <a id="add-access-policy"></a> Add an access policy to your Azure Key Vault instance

1. From the Azure portal, go to the Azure Key Vault instance that you plan to use to host your encryption keys. Select **Access Policies** from the left menu:

   :::image type="content" source="./media/how-to-setup-cmk/portal-akv-ap.png" alt-text="Access policies from the left menu":::

1. Select **+ Add Access Policy**.

1. Under the **Key permissions** drop-down menu, select **Get**, **Unwrap Key**, and **Wrap Key** permissions:

   :::image type="content" source="./media/how-to-setup-cmk/portal-akv-add-ap-perm2.png" alt-text="Selecting the right permissions":::

1. Under **Select principal**, select **None selected**.

1. Search for **Azure Cosmos DB** principal and select it (to make it easier to find, you can also search by principal ID: `a232010e-820c-4083-83bb-3ace5fc29d0b` for any Azure region except Azure Government regions where the principal ID is `57506a73-e302-42a9-b869-6f12d9ec29e9`). If the **Azure Cosmos DB** principal isn't in the list, you might need to re-register the **Microsoft.DocumentDB** resource provider as described in the [Register the resource provider](#register-resource-provider) section of this article.

   > [!NOTE]
   > This registers the Azure Cosmos DB first-party-identity in your Azure Key Vault access policy. To replace this first-party identity by your Azure Cosmos DB account managed identity, see [Using a managed identity in the Azure Key Vault access policy](#using-managed-identity).

1. Choose **Select** at the bottom. 

   :::image type="content" source="./media/how-to-setup-cmk/portal-akv-add-ap.png" alt-text="Select the Azure Cosmos DB principal":::

1. Select **Add** to add the new access policy.

1. Select **Save** on the Key Vault instance to save all changes.

## Generate a key in Azure Key Vault

1. From the Azure portal, go the Azure Key Vault instance that you plan to use to host your encryption keys. Then, select **Keys** from the left menu:

   :::image type="content" source="./media/how-to-setup-cmk/portal-akv-keys.png" alt-text="Keys entry from the left menu":::

1. Select **Generate/Import**, provide a name for the new key, and select an RSA key size. A minimum of 3072 is recommended for best security. Then select **Create**:

   :::image type="content" source="./media/how-to-setup-cmk/portal-akv-gen.png" alt-text="Create a new key":::

1. After the key is created, select the newly created key and then its current version.

1. Copy the key's **Key Identifier**, except the part after the last forward slash:

   :::image type="content" source="./media/how-to-setup-cmk/portal-akv-keyid.png" alt-text="Copying the key's key identifier":::

## Create a new Azure Cosmos account

### Using the Azure portal

When you create a new Azure Cosmos DB account from the Azure portal, choose **Customer-managed key** in the **Encryption** step. In the **Key URI** field, paste the URI/key identifier of the Azure Key Vault key that you copied from the previous step:

:::image type="content" source="./media/how-to-setup-cmk/portal-cosmos-enc.png" alt-text="Setting CMK parameters in the Azure portal":::

### <a id="using-powershell"></a> Using Azure PowerShell

When you create a new Azure Cosmos DB account with PowerShell:

- Pass the URI of the Azure Key Vault key copied earlier under the **keyVaultKeyUri** property in **PropertyObject**.

- Use **2019-12-12** or later as the API version.

> [!IMPORTANT]
> You must set the `locations` property explicitly for the account to be successfully created with customer-managed keys.

```powershell
$resourceGroupName = "myResourceGroup"
$accountLocation = "West US 2"
$accountName = "mycosmosaccount"

$failoverLocations = @(
    @{ "locationName"="West US 2"; "failoverPriority"=0 }
)

$CosmosDBProperties = @{
    "databaseAccountOfferType"="Standard";
    "locations"=$failoverLocations;
    "keyVaultKeyUri" = "https://<my-vault>.vault.azure.net/keys/<my-key>";
}

New-AzResource -ResourceType "Microsoft.DocumentDb/databaseAccounts" `
    -ApiVersion "2019-12-12" -ResourceGroupName $resourceGroupName `
    -Location $accountLocation -Name $accountName -PropertyObject $CosmosDBProperties
```

After the account has been created, you can verify that customer-managed keys have been enabled by fetching the URI of the Azure Key Vault key:

```powershell
Get-AzResource -ResourceGroupName $resourceGroupName -Name $accountName `
    -ResourceType "Microsoft.DocumentDb/databaseAccounts" `
    | Select-Object -ExpandProperty Properties `
    | Select-Object -ExpandProperty keyVaultKeyUri
```

### Using an Azure Resource Manager template

When you create a new Azure Cosmos account through an Azure Resource Manager template:

- Pass the URI of the Azure Key Vault key that you copied earlier under the **keyVaultKeyUri** property in the **properties** object.

- Use **2019-12-12** or later as the API version.

> [!IMPORTANT]
> You must set the `locations` property explicitly for the account to be successfully created with customer-managed keys.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "accountName": {
            "type": "string"
        },
        "location": {
            "type": "string"
        },
        "keyVaultKeyUri": {
            "type": "string"
        }
    },
    "resources": 
    [
        {
            "type": "Microsoft.DocumentDB/databaseAccounts",
            "name": "[parameters('accountName')]",
            "apiVersion": "2019-12-12",
            "kind": "GlobalDocumentDB",
            "location": "[parameters('location')]",
            "properties": {
                "locations": [ 
                    {
                        "locationName": "[parameters('location')]",
                        "failoverPriority": 0,
                        "isZoneRedundant": false
                    }
                ],
                "databaseAccountOfferType": "Standard",
                "keyVaultKeyUri": "[parameters('keyVaultKeyUri')]"
            }
        }
    ]
}
```

Deploy the template with the following PowerShell script:

```powershell
$resourceGroupName = "myResourceGroup"
$accountName = "mycosmosaccount"
$accountLocation = "West US 2"
$keyVaultKeyUri = "https://<my-vault>.vault.azure.net/keys/<my-key>"

New-AzResourceGroupDeployment `
    -ResourceGroupName $resourceGroupName `
    -TemplateFile "deploy.json" `
    -accountName $accountName `
    -location $accountLocation `
    -keyVaultKeyUri $keyVaultKeyUri
```

### <a id="using-azure-cli"></a> Using the Azure CLI

When you create a new Azure Cosmos account through the Azure CLI, pass the URI of the Azure Key Vault key that you copied earlier under the `--key-uri` parameter.

```azurecli-interactive
resourceGroupName='myResourceGroup'
accountName='mycosmosaccount'
keyVaultKeyUri = 'https://<my-vault>.vault.azure.net/keys/<my-key>'

az cosmosdb create \
    -n $accountName \
    -g $resourceGroupName \
    --locations regionName='West US 2' failoverPriority=0 isZoneRedundant=False \
    --key-uri $keyVaultKeyUri
```

After the account has been created, you can verify that customer-managed keys have been enabled by fetching the URI of the Azure Key Vault key:

```azurecli-interactive
az cosmosdb show \
    -n $accountName \
    -g $resourceGroupName \
    --query keyVaultKeyUri
```

## <a id="using-managed-identity"></a> Using a managed identity in the Azure Key Vault access policy

This access policy ensures that your encryption keys can be accessed by your Azure Cosmos DB account. This is done by granting access to a specific Azure Active Directory (AD) identity. Two types of identities are supported:

- Azure Cosmos DB's first-party identity can be used to grant access to the Azure Cosmos DB service.
- Your Azure Cosmos DB account's [managed identity](how-to-setup-managed-identity.md) can be used to grant access to your account specifically.

### To use a system-assigned managed identity

Because a system-assigned managed identity can only be retrieved after the creation of your account, you still need to initially create your account using the first-party identity, as described [above](#add-access-policy). Then:

1.	If this wasn't done during account creation, [enable a system-assigned managed identity](./how-to-setup-managed-identity.md#add-a-system-assigned-identity) on your account and copy the `principalId` that got assigned.

1.	Add a new access policy to your Azure Key Vault account just as described [above](#add-access-policy), but using the `principalId` you copied at the previous step instead of Azure Cosmos DB's first-party identity.

1.	Update your Azure Cosmos DB account to specify that you want to use the system-assigned managed identity when accessing your encryption keys in Azure Key Vault. You can do this:

    - by specifying this property in your account's Azure Resource Manager template:

    ```json
    {
        "type": " Microsoft.DocumentDB/databaseAccounts",
        "properties": {
            "defaultIdentity": "SystemAssignedIdentity",
            // ...
        },
        // ...
    }
    ```

    - by updating your account with the Azure CLI:

    ```azurecli
        resourceGroupName='myResourceGroup'
        accountName='mycosmosaccount'

        az cosmosdb update --resource-group $resourceGroupName --name $accountName --default-identity "SystemAssignedIdentity"
    ```
  
1.	Optionally, you can then remove the Azure Cosmos DB first-party identity from your Azure Key Vault access policy.

### To use a user-assigned managed identity

> [!IMPORTANT]
> When using a user-assigned managed identity, firewall rules on the Azure Key Vault account aren't currently supported. You must keep your Azure Key Vault account accessible from all networks.

1.	When creating the new access policy in your Azure Key Vault account as described [above](#add-access-policy), use the `Object ID` of the managed identity you wish to use instead of Azure Cosmos DB's first-party identity.

1.	When creating your Azure Cosmos DB account, you must enable the user-assigned managed identity and specify that you want to use this identity when accessing your encryption keys in Azure Key Vault. You can do this:

    - in an Azure Resource Manager template:

    ```json
    {
        "type": "Microsoft.DocumentDB/databaseAccounts",
        "identity": {
            "type": "UserAssigned",
            "userAssignedIdentities": {
                "<identity-resource-id>": {}
            }
        },
        // ...
        "properties": {
            "defaultIdentity": "UserAssignedIdentity=<identity-resource-id>"
            "keyVaultKeyUri": "<key-vault-key-uri>"
            // ...
        }
    }
    ```

    - with the Azure CLI:

    ```azurecli
    resourceGroupName='myResourceGroup'
    accountName='mycosmosaccount'
    keyVaultKeyUri = 'https://<my-vault>.vault.azure.net/keys/<my-key>'

    az cosmosdb create \
        -n $accountName \
        -g $resourceGroupName \
        --key-uri $keyVaultKeyUri
        --assign-identity <identity-resource-id>
        --default-identity "UserAssignedIdentity=<identity-resource-id>"  
    ```

## Key rotation

Rotating the customer-managed key used by your Azure Cosmos account can be done in two ways.

- Create a new version of the key currently used from Azure Key Vault:

  :::image type="content" source="./media/how-to-setup-cmk/portal-akv-rot.png" alt-text="Create a new key version":::

- Swap the key currently used with a totally different one by updating the key URI on your account. From the Azure portal, go to your Azure Cosmos account and select **Data Encryption** from the left menu:

    :::image type="content" source="./media/how-to-setup-cmk/portal-data-encryption.png" alt-text="The Data Encryption menu entry":::

    Then, replace the **Key URI** with the new key you want to use and select **Save**:

    :::image type="content" source="./media/how-to-setup-cmk/portal-key-swap.png" alt-text="Update the key URI":::

    Here's how to do achieve the same result in PowerShell:

    ```powershell
    $resourceGroupName = "myResourceGroup"
    $accountName = "mycosmosaccount"
    $newKeyUri = "https://<my-vault>.vault.azure.net/keys/<my-new-key>"
    
    $account = Get-AzResource -ResourceGroupName $resourceGroupName -Name $accountName `
        -ResourceType "Microsoft.DocumentDb/databaseAccounts"
    
    $account.Properties.keyVaultKeyUri = $newKeyUri
    
    $account | Set-AzResource -Force
    ```

The previous key or key version can be disabled after the [Azure Key Vault audit logs](../key-vault/general/logging.md) don't show activity from Azure Cosmos DB on that key or key version anymore. No more activity should take place on the previous key or key version after 24 hours of key rotation.
    
## Error handling

When using customer-managed keys in Azure Cosmos DB, if there are any errors, Azure Cosmos DB returns the error details along with a HTTP sub-status code in the response. You can use this sub-status code to debug the root cause of the issue. See the [HTTP Status Codes for Azure Cosmos DB](/rest/api/cosmos-db/http-status-codes-for-cosmosdb) article to get the list of supported HTTP sub-status codes.

## Frequently asked questions

### Is there an additional charge to enable customer-managed keys?

No, there's no charge to enable this feature.

### How do customer-managed keys impact capacity planning?

When using customer-managed keys, [Request Units](./request-units.md) consumed by your database operations see an increase to reflect the additional processing required to perform encryption and decryption of your data. This may lead to slightly higher utilization of your provisioned capacity. Use the table below for guidance:

| Operation type | Request Unit increase |
|---|---|
| Point-reads (fetching items by their ID) | + 5% per operation |
| Any write operation | + 6% per operation <br/> Approximately + 0.06 RU per indexed property |
| Queries, reading change feed, or conflict feed | + 15% per operation |

### What data gets encrypted with the customer-managed keys?

All the data stored in your Azure Cosmos account is encrypted with the customer-managed keys, except for the following metadata:

- The names of your Azure Cosmos DB [accounts, databases, and containers](./account-databases-containers-items.md#elements-in-an-azure-cosmos-account)

- The names of your [stored procedures](./stored-procedures-triggers-udfs.md)

- The property paths declared in your [indexing policies](./index-policy.md)

- The values of your containers' [partition keys](./partitioning-overview.md)

### Are customer-managed keys supported for existing Azure Cosmos accounts?

This feature is currently available only for new accounts.

### Is it possible to use customer-managed keys in conjunction with the Azure Cosmos DB [analytical store](analytical-store-introduction.md)?

Yes, Azure Synapse Link only supports configuring customer-managed keys using your Azure Cosmos DB account's managed identity. You must [use your Azure Cosmos DB account's managed identity](#using-managed-identity) in your Azure Key Vault access policy before [enabling Azure Synapse Link](configure-synapse-link.md#enable-synapse-link) on your account.

### Is there a plan to support finer granularity than account-level keys?

Not currently, but container-level keys are being considered.

### How can I tell if customer-managed keys are enabled on my Azure Cosmos account?

From the Azure portal, go to your Azure Cosmos account and watch for the **Data Encryption** entry in the left menu; if this entry exists, customer-managed keys are enabled on your account:

:::image type="content" source="./media/how-to-setup-cmk/portal-data-encryption.png" alt-text="The Data Encryption menu entry":::

You can also programmatically fetch the details of your Azure Cosmos account and look for the presence of the `keyVaultKeyUri` property. See above for ways to do that [in PowerShell](#using-powershell) and [using the Azure CLI](#using-azure-cli).

### How do customer-managed keys affect periodic backups?

Azure Cosmos DB takes [regular and automatic backups](./online-backup-and-restore.md) of the data stored in your account. This operation backs up the encrypted data.

The following conditions are necessary to successfully restore a periodic backup:
- The encryption key that you used at the time of the backup is required and must be available in Azure Key Vault. This means that no revocation was made and the version of the key that was used at the time of the backup is still enabled.
- If you [used a system-assigned managed identity in the Azure Key Vault access policy](#to-use-a-system-assigned-managed-identity) of the source account, you must temporarily grant access to the Azure Cosmos DB first-party identity in that access policy as described [here](#add-access-policy) before restoring your data. Once the data is fully restored to the target account, you can remove the first-party identity from the Key Vault access policy and set your desired identity configuration.

### How do I revoke an encryption key?

Key revocation is done by disabling the latest version of the key:

:::image type="content" source="./media/how-to-setup-cmk/portal-akv-rev2.png" alt-text="Disable a key's version":::

Alternatively, to revoke all keys from an Azure Key Vault instance, you can delete the access policy granted to the Azure Cosmos DB principal:

:::image type="content" source="./media/how-to-setup-cmk/portal-akv-rev.png" alt-text="Deleting the access policy for the Azure Cosmos DB principal":::

### What operations are available after a customer-managed key is revoked?

The only operation possible when the encryption key has been revoked is account deletion.

## Next steps

- Learn more about [data encryption in Azure Cosmos DB](./database-encryption-at-rest.md).
- Get an overview of [secure access to data in Cosmos DB](secure-access-to-data.md).
