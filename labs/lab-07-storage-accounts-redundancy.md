# Lab 7 — Create and Configure Storage Accounts and Redundancy

Maps to AZ-104 objective **Implement and manage storage → Configure access to storage** (create and configure storage accounts; configure redundancy — LRS/ZRS/GRS/RA-GRS; configure storage account encryption; configure object replication).

You will create general-purpose v2 storage accounts, walk through every redundancy option the exam tests, inspect and configure encryption (Microsoft-managed and customer-managed keys), and set up object replication between a source and destination account so blobs copy asynchronously across accounts.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Azure Storage (general-purpose v2 accounts), Blob storage, Azure Key Vault (for CMK), object replication.

---

## Step 1 — Create a resource group

**Portal:** In the search bar type **Resource groups → + Create**. Set **Subscription**, **Resource group** = `rg-az104-lab07`, **Region** = `East US`, then **Review + create → Create**.

```bash
az group create --name rg-az104-lab07 --location eastus
```

```powershell
New-AzResourceGroup -Name rg-az104-lab07 -Location eastus
```

## Step 2 — Create a Locally-Redundant (LRS) storage account

Storage account names are **globally unique, 3–24 chars, lowercase letters and numbers only**. We use a `$RANDOM` suffix so the name is unique in the whole world.

**Portal:** **Storage accounts → + Create**. Resource group `rg-az104-lab07`; **Storage account name** (globally unique, e.g. `stlab07src12345`); **Region** East US; **Primary service** Azure Blob Storage; **Performance** Standard; **Redundancy** = *Locally-redundant storage (LRS)*. **Review + create → Create**.

```bash
# Save one unique source name and reuse it for the rest of the lab
SRC=stlab07src$RANDOM
echo "Source account: $SRC"   # note this value

az storage account create \
  --name $SRC \
  --resource-group rg-az104-lab07 \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --access-tier Hot
# LRS keeps 3 synchronous copies inside ONE datacenter (cheapest, no zone/region protection)
```

```powershell
$Src = "stlab07src$(Get-Random -Maximum 99999)"
Write-Host "Source account: $Src"
New-AzStorageAccount -ResourceGroupName rg-az104-lab07 -Name $Src `
  -Location eastus -SkuName Standard_LRS -Kind StorageV2 -AccessTier Hot
```

## Step 3 — Compare and change redundancy (LRS → ZRS → GRS → RA-GRS)

The **redundancy** (SKU) can be changed live for most conversions. Know these for the exam:

| SKU | Copies | Protection |
|-----|--------|------------|
| `Standard_LRS` | 3 in one datacenter | Hardware failure |
| `Standard_ZRS` | 3 across availability zones | Zone failure |
| `Standard_GRS` | LRS in primary + LRS in paired region (async) | Regional disaster |
| `Standard_RAGRS` | GRS + **read access** to secondary | Regional disaster + read fallback |
| `Standard_GZRS` / `Standard_RAGZRS` | ZRS in primary + LRS in secondary | Zone + region |

**Portal:** Open the account → **Data management → Redundancy** → choose a new option → **Save**. (Some conversions such as LRS→ZRS may be shown as a *conversion request*.)

```bash
# Switch to geo-redundant with read access to the secondary region
az storage account update --name $SRC --resource-group rg-az104-lab07 --sku Standard_RAGRS

# Confirm the SKU and view the secondary (read) endpoints
az storage account show --name $SRC --resource-group rg-az104-lab07 \
  --query "{sku:sku.name, primary:primaryEndpoints.blob, secondary:secondaryEndpoints.blob}" -o table
```

```powershell
Set-AzStorageAccount -ResourceGroupName rg-az104-lab07 -Name $Src -SkuName Standard_RAGRS
(Get-AzStorageAccount -ResourceGroupName rg-az104-lab07 -Name $Src).Sku.Name
```

## Step 4 — Inspect and configure storage account encryption

All Azure Storage is **encrypted at rest by default** with Microsoft-managed keys (256-bit AES, SSE). You can also enable **infrastructure encryption** (double encryption) and switch to **customer-managed keys (CMK)**.

**Portal:** Account → **Security + networking → Encryption**. See *Encryption type* = Microsoft-managed keys. Toggle to *Customer-managed keys* and select a Key Vault + key.

```bash
# View current encryption settings
az storage account show --name $SRC --resource-group rg-az104-lab07 \
  --query "encryption" -o json
# services.blob.enabled=true and keySource=Microsoft.Storage means SSE with MS-managed keys
```

Enable customer-managed keys with a Key Vault (CMK):

```bash
KV=kvlab07$RANDOM
az keyvault create --name $KV --resource-group rg-az104-lab07 --location eastus \
  --enable-purge-protection true --enable-rbac-authorization false
az keyvault key create --vault-name $KV --name storagecmk --kind RSA --size 2048

# Give the storage account a system-assigned identity, then grant it key access
az storage account update --name $SRC --resource-group rg-az104-lab07 --assign-identity
PRINCIPAL=$(az storage account show --name $SRC -g rg-az104-lab07 --query identity.principalId -o tsv)
az keyvault set-policy --name $KV --object-id $PRINCIPAL \
  --key-permissions get wrapKey unwrapKey

# Point the storage account at the CMK
az storage account update --name $SRC --resource-group rg-az104-lab07 \
  --encryption-key-source Microsoft.Keyvault \
  --encryption-key-vault "https://$KV.vault.azure.net/" \
  --encryption-key-name storagecmk
```

## Step 5 — Create the destination account for object replication

Object replication needs a **second** account. Both accounts require **blob versioning** and **change feed** enabled.

**Portal:** Create a second storage account `stlab07dst…` the same way as Step 2.

```bash
DST=stlab07dst$RANDOM
echo "Destination account: $DST"
az storage account create --name $DST --resource-group rg-az104-lab07 \
  --location westus --sku Standard_LRS --kind StorageV2
```

## Step 6 — Enable versioning and change feed on both accounts

**Portal:** On each account → **Data protection** → tick **Enable versioning for blobs** and **Enable blob change feed** → **Save**.

```bash
for ACC in $SRC $DST; do
  az storage account blob-service-properties update \
    --account-name $ACC --resource-group rg-az104-lab07 \
    --enable-versioning true --enable-change-feed true
done
```

## Step 7 — Create source and destination containers with a blob

**Portal:** In each account → **Containers → + Container**. Name both `orders`.

```bash
az storage container create --account-name $SRC --name orders --auth-mode login
az storage container create --account-name $DST --name orders --auth-mode login

echo "replicate me" > sample.txt
az storage blob upload --account-name $SRC --container-name orders \
  --name sample.txt --file sample.txt --auth-mode login
```

## Step 8 — Configure an object replication policy

**Portal:** On the **source** account → **Data management → Object replication → Set up replication rules**. Pick the destination account/subscription, map source container `orders` → destination `orders`, optionally add a prefix filter, then **Create**.

```bash
# Create the replication policy on the DESTINATION account first (it returns a policy id),
# then apply the returned policy to the SOURCE account.
az storage account or-policy create \
  --account-name $DST --resource-group rg-az104-lab07 \
  --source-account $SRC --destination-account $DST \
  --source-container orders --destination-container orders \
  --min-creation-time '2024-01-01T00:00:00Z'

# List policies to confirm
az storage account or-policy list --account-name $SRC -g rg-az104-lab07 -o table
```

Replication is asynchronous. After a few minutes, confirm the blob appears in the destination:

```bash
az storage blob list --account-name $DST --container-name orders \
  --auth-mode login -o table
```

## Clean up

```bash
az group delete --name rg-az104-lab07 --yes --no-wait
```

Key Vault purge protection means the vault name may be soft-deleted for a retention period; that is expected and incurs no cost.

## References

- Storage account overview: https://learn.microsoft.com/azure/storage/common/storage-account-overview
- Data redundancy options: https://learn.microsoft.com/azure/storage/common/storage-redundancy
- Storage encryption (SSE / CMK): https://learn.microsoft.com/azure/storage/common/storage-service-encryption
- Object replication for block blobs: https://learn.microsoft.com/azure/storage/blobs/object-replication-overview

## What you learned

- Create a general-purpose v2 storage account with a globally unique name via Portal, CLI, and PowerShell.
- Distinguish and switch between LRS, ZRS, GRS, RA-GRS (and GZRS) redundancy tiers and what each protects against.
- Verify default SSE encryption and reconfigure a storage account to use customer-managed keys backed by Azure Key Vault.
- Enable blob versioning and change feed as prerequisites for object replication.
- Build an object replication policy that asynchronously copies blobs from a source account to a destination account in another region.
