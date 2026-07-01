# Lab 5 — Manage Resource Groups, Locks and Tags

Maps to AZ-104 objective **Manage Azure identities and governance → Manage resource groups and subscriptions** (create and manage resource groups, move resources between groups, configure resource locks, apply and manage tags).

You will create a resource group, deploy a resource into it, apply and read tags, move a resource between groups, and protect resources with **CanNotDelete** and **ReadOnly** locks.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Azure Resource Manager (resource groups, tags, resource locks, resource move), a storage account as the test resource.

---

## Step 1 — Create two resource groups

**Portal:** **Resource groups → + Create** twice: `rg-az104-lab05` and `rg-az104-lab05-archive`, same region, **Review + create**.

```bash
az login
LOCATION="southeastasia"
RG="rg-az104-lab05"
RG2="rg-az104-lab05-archive"

az group create --name "$RG"  --location "$LOCATION"
az group create --name "$RG2" --location "$LOCATION"
az group list --query "[?starts_with(name,'rg-az104-lab05')].{name:name, location:location}" -o table
```

```powershell
New-AzResourceGroup -Name "rg-az104-lab05" -Location "southeastasia"
New-AzResourceGroup -Name "rg-az104-lab05-archive" -Location "southeastasia"
```

## Step 2 — Deploy a resource to tag, move and lock

**Portal:** Inside `rg-az104-lab05` **+ Create → Storage account**, unique name, **Review + create**.

```bash
SA="stlab05$RANDOM"
az storage account create --name "$SA" --resource-group "$RG" \
  --location "$LOCATION" --sku Standard_LRS
SA_ID=$(az storage account show --name "$SA" --resource-group "$RG" --query id -o tsv)
echo "$SA_ID"
```

## Step 3 — Apply tags to the resource group

**Portal:** **rg-az104-lab05 → Tags**, add `CostCenter=1001`, `Environment=Dev`, `Owner=angch`, **Save**.

```bash
# set tags on the resource group (this REPLACES all existing tags)
az tag create --resource-id "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG" \
  --tags CostCenter=1001 Environment=Dev Owner=angch

# read tags back
az group show --name "$RG" --query tags
```

```powershell
Set-AzResourceGroup -Name "rg-az104-lab05" `
  -Tag @{ CostCenter="1001"; Environment="Dev"; Owner="angch" }
(Get-AzResourceGroup -Name "rg-az104-lab05").Tags
```

> Tags on a resource group are **not** inherited by the resources inside it. To enforce/inherit tags, use an Azure Policy `Modify` effect (see Lab 4).

## Step 4 — Apply and update tags on an individual resource

**Portal:** **Storage account → Tags**, add `Environment=Dev`, `Project=AZ104`, **Save**.

```bash
# add/merge a tag onto the storage account without wiping others
az tag update --resource-id "$SA_ID" --operation Merge \
  --tags Environment=Dev Project=AZ104

# change one tag value with Merge (does not remove other tags)
az tag update --resource-id "$SA_ID" --operation Merge --tags Environment=Test

# remove a single tag
az tag update --resource-id "$SA_ID" --operation Delete --tags Project=AZ104

# list resources by tag across the subscription
az resource list --tag Environment=Test -o table
```

> `--operation Merge` adds/overwrites the named tags and keeps the rest; `Replace` overwrites all tags; `Delete` removes the named ones.

## Step 5 — Move a resource to another resource group

**Portal:** **Storage account → Move → Move to another resource group**, pick `rg-az104-lab05-archive`, tick the confirmation, **OK**. The move can take a few minutes.

```bash
DEST_ID=$(az group show --name "$RG2" --query id -o tsv)

# validate the move first (checks the resource type supports moving)
az resource invoke-action --action validateMoveResources \
  --ids "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG" \
  --request-body "{\"resources\":[\"$SA_ID\"],\"targetResourceGroup\":\"$DEST_ID\"}" 2>/dev/null

# perform the move
az resource move --destination-group "$RG2" --ids "$SA_ID"

# confirm it now lives in the archive RG
az storage account show --name "$SA" --resource-group "$RG2" --query "{name:name, rg:resourceGroup}"
SA_ID=$(az storage account show --name "$SA" --resource-group "$RG2" --query id -o tsv)
```

> Not every resource type supports move, and locks on either RG block moves. Remove locks before moving.

## Step 6 — Apply a CanNotDelete lock

**Portal:** **rg-az104-lab05-archive → Locks → + Add**, **Lock type = Delete**, Name `lock-no-delete`, **OK**.

```bash
# lock the resource group so nothing in it can be deleted
az lock create --name "lock-no-delete" --lock-type CanNotDelete \
  --resource-group "$RG2" \
  --notes "Protect archived resources from deletion"

az lock list --resource-group "$RG2" -o table
```

```powershell
New-AzResourceLock -LockName "lock-no-delete" -LockLevel CanNotDelete `
  -ResourceGroupName "rg-az104-lab05-archive" -LockNotes "Protect archived resources" -Force
```

## Step 7 — Test the lock, then apply a ReadOnly lock

**Portal:** Try deleting the storage account — the Portal blocks it citing the lock.

```bash
# this DELETE should fail because of the CanNotDelete lock
az storage account delete --name "$SA" --resource-group "$RG2" --yes
# expected: "ScopeLocked" error

# apply a ReadOnly lock on the storage account itself (blocks modify AND delete)
az lock create --name "lock-readonly" --lock-type ReadOnly \
  --resource-group "$RG2" \
  --resource-name "$SA" \
  --resource-type "Microsoft.Storage/storageAccounts" \
  --namespace "Microsoft.Storage"
```

> **CanNotDelete** = can read/modify but not delete. **ReadOnly** = can read only (no modify, no delete). Locks are inherited downward: a lock on a subscription or RG applies to everything within.

## Step 8 — Manage subscription-level settings (context)

**Portal:** **Subscriptions → your subscription → Resource providers** to register/unregister providers (e.g. `Microsoft.Storage`), and **Access control (IAM)** for subscription roles.

```bash
# view and register a resource provider (needed before creating that resource type)
az provider list --query "[?registrationState=='Registered'].namespace" -o tsv | head
az provider register --namespace "Microsoft.Storage"
az provider show --namespace "Microsoft.Storage" --query registrationState
```

## Clean up

Locks block deletion, so remove them first.

```bash
# remove locks, then the resource groups
az lock delete --name "lock-readonly" --resource-group "$RG2" \
  --resource-name "$SA" --resource-type "Microsoft.Storage/storageAccounts" \
  --namespace "Microsoft.Storage"
az lock delete --name "lock-no-delete" --resource-group "$RG2"

az group delete --name rg-az104-lab05 --yes --no-wait
az group delete --name rg-az104-lab05-archive --yes --no-wait
```

## References

- Manage Azure resource groups — https://learn.microsoft.com/azure/azure-resource-manager/management/manage-resource-groups-portal
- Move resources to a new resource group or subscription — https://learn.microsoft.com/azure/azure-resource-manager/management/move-resource-group-and-subscription
- Lock resources to prevent changes — https://learn.microsoft.com/azure/azure-resource-manager/management/lock-resources
- Use tags to organize your Azure resources — https://learn.microsoft.com/azure/azure-resource-manager/management/tag-resources

## What you learned

- Create and manage **resource groups** and register **resource providers** at the subscription.
- Apply, merge, update and delete **tags** on both resource groups and individual resources, and query by tag.
- Understand that RG tags are **not inherited** by resources (use Policy Modify to enforce).
- **Move** a resource between resource groups and validate the move first.
- Apply **CanNotDelete** and **ReadOnly** locks, understand their inheritance, and why locks must be removed before moving or deleting.
