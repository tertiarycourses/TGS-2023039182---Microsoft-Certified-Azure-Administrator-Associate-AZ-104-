# Lab 9 — Configure Azure Blob Storage: Tiers, Lifecycle, Versioning

Maps to AZ-104 objective **Implement and manage storage → Configure Azure Blob Storage** (create and configure a Blob container; configure access tiers — hot/cool/cold/archive; configure soft delete for blobs and containers; configure blob versioning; configure blob lifecycle management).

You will create a blob container, upload blobs and move them between the hot, cool, cold, and archive access tiers, turn on soft delete for both blobs and containers, enable blob versioning, and author a JSON lifecycle management policy that ages blobs down automatically.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Azure Blob Storage, access tiers, blob/container soft delete, blob versioning, lifecycle management policies.

---

## Step 1 — Create the resource group and storage account

Blob access tiers and lifecycle management require a **general-purpose v2** (or Blob) account.

```bash
az group create --name rg-az104-lab09 --location eastus

SA=stlab09$RANDOM
echo "Storage account: $SA"   # globally unique lowercase name
az storage account create --name $SA --resource-group rg-az104-lab09 \
  --location eastus --sku Standard_LRS --kind StorageV2 --access-tier Hot
```

```powershell
New-AzResourceGroup -Name rg-az104-lab09 -Location eastus
$Sa = "stlab09$(Get-Random -Maximum 99999)"
New-AzStorageAccount -ResourceGroupName rg-az104-lab09 -Name $Sa -Location eastus -SkuName Standard_LRS -Kind StorageV2 -AccessTier Hot
```

## Step 2 — Create a blob container

**Portal:** Account → **Data storage → Containers → + Container** → name `media`, Public access level *Private* → **Create**.

```bash
az storage container create --account-name $SA --name media --auth-mode login
```

```powershell
$Ctx = (Get-AzStorageAccount -ResourceGroupName rg-az104-lab09 -Name $Sa).Context
New-AzStorageContainer -Name media -Context $Ctx -Permission Off
```

## Step 3 — Upload blobs in different tiers

Blob tiers (cheapest storage → highest access cost/latency): **Hot → Cool → Cold → Archive**. Archive is offline and must be *rehydrated* before reading.

**Portal:** Container `media` → **Upload** → expand **Advanced** → choose **Access tier**.

```bash
echo "hot data"  > hot.txt
echo "cool data" > cool.txt

az storage blob upload --account-name $SA --container-name media \
  --name hot.txt  --file hot.txt  --tier Hot  --auth-mode login
az storage blob upload --account-name $SA --container-name media \
  --name cool.txt --file cool.txt --tier Cool --auth-mode login
```

## Step 4 — Change the access tier of an existing blob

**Portal:** Container → select a blob → **Change tier** → pick Hot/Cool/Cold/Archive → **Save**.

```bash
# Move cool.txt down to Cold
az storage blob set-tier --account-name $SA --container-name media \
  --name cool.txt --tier Cold --auth-mode login

# Archive hot.txt (goes offline; reads require rehydration to Hot/Cool)
az storage blob set-tier --account-name $SA --container-name media \
  --name hot.txt --tier Archive --auth-mode login

# Verify tiers
az storage blob list --account-name $SA --container-name media --auth-mode login \
  --query "[].{name:name, tier:properties.blobTier}" -o table
```

## Step 5 — Enable blob versioning

Versioning automatically keeps previous versions of a blob every time it is overwritten.

**Portal:** Account → **Data protection** → tick **Enable versioning for blobs** → **Save**.

```bash
az storage account blob-service-properties update \
  --account-name $SA --resource-group rg-az104-lab09 \
  --enable-versioning true

# Overwrite a blob to generate a new version
echo "hot data v2" > hot.txt
az storage blob upload --account-name $SA --container-name media \
  --name hot.txt --file hot.txt --overwrite --auth-mode login

# List versions
az storage blob list --account-name $SA --container-name media --include v \
  --auth-mode login --query "[?name=='hot.txt'].{name:name, version:versionId, current:isCurrentVersion}" -o table
```

## Step 6 — Enable soft delete for blobs and containers

Soft delete lets you recover deleted blobs (and whole containers) within a retention window.

**Portal:** Account → **Data protection** → tick **Enable soft delete for blobs** (set days, e.g. 7) and **Enable soft delete for containers** (set days) → **Save**.

```bash
# Blob soft delete (7-day retention)
az storage account blob-service-properties update \
  --account-name $SA --resource-group rg-az104-lab09 \
  --enable-delete-retention true --delete-retention-days 7

# Container soft delete (7-day retention)
az storage account blob-service-properties update \
  --account-name $SA --resource-group rg-az104-lab09 \
  --enable-container-delete-retention true --container-delete-retention-days 7
```

Test recovery:

```bash
# Delete then list soft-deleted blobs, then undelete
az storage blob delete --account-name $SA --container-name media --name cool.txt --auth-mode login
az storage blob list --account-name $SA --container-name media --include d \
  --auth-mode login -o table
az storage blob undelete --account-name $SA --container-name media --name cool.txt --auth-mode login
```

## Step 7 — Author a lifecycle management policy

Lifecycle rules run daily to tier-down or delete blobs based on last-modified/last-accessed age. The policy is a **JSON** document.

**Portal:** Account → **Data management → Lifecycle management → + Add a rule** → set conditions (e.g. move to Cool after 30 days, Archive after 90, delete after 365) → **Add**.

```bash
cat > policy.json <<'EOF'
{
  "rules": [
    {
      "enabled": true,
      "name": "tier-down-and-expire",
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": [ "blockBlob" ],
          "prefixMatch": [ "media/" ]
        },
        "actions": {
          "baseBlob": {
            "tierToCool":    { "daysAfterModificationGreaterThan": 30 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 90 },
            "delete":        { "daysAfterModificationGreaterThan": 365 }
          },
          "snapshot": { "delete": { "daysAfterCreationGreaterThan": 90 } },
          "version":  { "delete": { "daysAfterCreationGreaterThan": 90 } }
        }
      }
    }
  ]
}
EOF

az storage account management-policy create \
  --account-name $SA --resource-group rg-az104-lab09 \
  --policy @policy.json
```

## Step 8 — Verify the lifecycle policy

**Portal:** **Lifecycle management → Code view** shows the same JSON you applied.

```bash
az storage account management-policy show \
  --account-name $SA --resource-group rg-az104-lab09 -o json
```

## Clean up

```bash
az group delete --name rg-az104-lab09 --yes --no-wait
```

## References

- Blob access tiers (hot/cool/cold/archive): https://learn.microsoft.com/azure/storage/blobs/access-tiers-overview
- Soft delete for blobs: https://learn.microsoft.com/azure/storage/blobs/soft-delete-blob-overview
- Blob versioning: https://learn.microsoft.com/azure/storage/blobs/versioning-overview
- Optimize costs by automating access tiers (lifecycle management): https://learn.microsoft.com/azure/storage/blobs/lifecycle-management-overview

## What you learned

- Create a blob container and upload blobs directly into the hot, cool, cold, and archive tiers.
- Change a blob's tier and understand that archive is offline and needs rehydration.
- Enable blob versioning so overwrites automatically preserve previous versions.
- Turn on soft delete for both blobs and containers and recover a deleted blob.
- Author and apply a JSON lifecycle management policy that tiers-down and expires blobs automatically.
