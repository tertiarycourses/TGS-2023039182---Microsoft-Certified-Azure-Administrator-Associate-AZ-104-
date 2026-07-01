# Lab 8 — Configure Access to Storage: Firewalls, SAS, Access Keys

Maps to AZ-104 objective **Implement and manage storage → Configure access to storage** (configure Azure Storage firewalls and virtual networks; create and use shared access signature (SAS) tokens; configure stored access policies; manage access keys; configure identity-based access with RBAC).

You will lock a storage account behind its network firewall, generate account and service SAS tokens with scoped permissions, tie a SAS to a stored access policy you can revoke, rotate access keys, and grant identity-based (Azure AD / Microsoft Entra) access using RBAC data roles.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Azure Storage, storage account firewall + virtual networks (service endpoints), SAS, stored access policies, access keys, Azure RBAC (Storage Blob Data roles).

---

## Step 1 — Create the resource group and storage account

```bash
az group create --name rg-az104-lab08 --location eastus

SA=stlab08$RANDOM
echo "Storage account: $SA"   # note this globally unique lowercase name
az storage account create --name $SA --resource-group rg-az104-lab08 \
  --location eastus --sku Standard_LRS --kind StorageV2
```

```powershell
New-AzResourceGroup -Name rg-az104-lab08 -Location eastus
$Sa = "stlab08$(Get-Random -Maximum 99999)"
New-AzStorageAccount -ResourceGroupName rg-az104-lab08 -Name $Sa -Location eastus -SkuName Standard_LRS -Kind StorageV2
```

## Step 2 — Create a container and a blob to protect

**Portal:** Account → **Containers → + Container** → name `docs`.

```bash
az storage container create --account-name $SA --name docs --auth-mode login
echo "secret report" > report.txt
az storage blob upload --account-name $SA --container-name docs \
  --name report.txt --file report.txt --auth-mode login
```

## Step 3 — Manage and rotate access keys

Every account has **two access keys** (`key1`, `key2`) so you can rotate one while the other stays live. A **connection string** bundles a key.

**Portal:** Account → **Security + networking → Access keys**. Click **Show**, copy a key, and use **Rotate key** to regenerate.

```bash
# List both keys
az storage account keys list --account-name $SA --resource-group rg-az104-lab08 -o table

# Rotate (regenerate) key1 — do this after moving clients to key2
az storage account keys renew --account-name $SA --resource-group rg-az104-lab08 --key primary

# Grab the current key into a variable for later SAS generation
KEY=$(az storage account keys list --account-name $SA -g rg-az104-lab08 --query "[0].value" -o tsv)
```

```powershell
New-AzStorageAccountKey -ResourceGroupName rg-az104-lab08 -Name $Sa -KeyName key1
```

## Step 4 — Generate an account SAS

An **account SAS** can grant access across multiple services (blob/file/queue/table). Scope it to the minimum permissions and a short expiry.

**Portal:** Account → **Security + networking → Shared access signature** → pick allowed services/resource types/permissions and an expiry → **Generate SAS and connection string**.

```bash
EXPIRY=$(date -u -d '2 hours' '+%Y-%m-%dT%H:%MZ' 2>/dev/null || date -u -v+2H '+%Y-%m-%dT%H:%MZ')

az storage account generate-sas \
  --account-name $SA --account-key $KEY \
  --services b --resource-types sco \
  --permissions rl --expiry $EXPIRY --https-only -o tsv
# services b=blob; resource-types s=service c=container o=object; permissions r=read l=list
```

## Step 5 — Generate a service (blob) SAS for one blob

A **service SAS** is scoped to a single service and can target one blob or container.

```bash
az storage blob generate-sas \
  --account-name $SA --account-key $KEY \
  --container-name docs --name report.txt \
  --permissions r --expiry $EXPIRY --https-only --full-uri -o tsv
# --full-uri returns a ready-to-use https URL you can paste into a browser to download report.txt
```

## Step 6 — Create a stored access policy and bind a SAS to it

A **stored access policy** lives on the container. A SAS tied to it inherits its expiry/permissions and can be **revoked centrally** by deleting the policy.

**Portal:** Container `docs` → **Settings → Access policy → + Add policy** (Stored access policies). Set identifier, permissions, expiry → **OK → Save**.

```bash
POLICYEXP=$(date -u -d '1 day' '+%Y-%m-%dT%H:%MZ' 2>/dev/null || date -u -v+1d '+%Y-%m-%dT%H:%MZ')

az storage container policy create \
  --account-name $SA --account-key $KEY \
  --container-name docs --name readonly \
  --permissions rl --expiry $POLICYEXP

# Issue a SAS that references the policy — no permissions/expiry of its own
az storage blob generate-sas \
  --account-name $SA --account-key $KEY \
  --container-name docs --name report.txt \
  --policy-name readonly --full-uri -o tsv

# Revoke ALL SAS issued under this policy by deleting it
az storage container policy delete --account-name $SA --account-key $KEY \
  --container-name docs --name readonly
```

## Step 7 — Configure the storage firewall and virtual network access

By default the account allows access from all networks. Restrict it to a specific VNet subnet (service endpoint) and specific public IPs.

**Portal:** Account → **Security + networking → Networking → Firewalls and virtual networks** → *Enabled from selected virtual networks and IP addresses* → **+ Add existing virtual network** and **+ Add your client IP**. Keep **Allow Azure services on the trusted services list** ticked → **Save**.

```bash
# Create a VNet/subnet with the storage service endpoint
az network vnet create --resource-group rg-az104-lab08 --name vnet-lab08 \
  --address-prefix 10.10.0.0/16 --subnet-name apps --subnet-prefix 10.10.1.0/24
az network vnet subnet update --resource-group rg-az104-lab08 --vnet-name vnet-lab08 \
  --name apps --service-endpoints Microsoft.Storage

# Set the account to deny by default, then add rules
az storage account update --name $SA --resource-group rg-az104-lab08 \
  --default-action Deny --bypass AzureServices

az storage account network-rule add --account-name $SA --resource-group rg-az104-lab08 \
  --vnet-name vnet-lab08 --subnet apps
az storage account network-rule add --account-name $SA --resource-group rg-az104-lab08 \
  --ip-address "$(curl -s ifconfig.me)"

az storage account network-rule list --account-name $SA --resource-group rg-az104-lab08 -o json
```

## Step 8 — Configure identity-based (Azure AD / Entra) access with RBAC

Instead of keys/SAS, grant your user a **data-plane RBAC role** and access with `--auth-mode login`. This is the recommended, keyless approach.

**Portal:** Account → **Access Control (IAM) → + Add → Add role assignment** → role **Storage Blob Data Contributor** → assign to your user → **Review + assign**.

```bash
USER=$(az ad signed-in-user show --query id -o tsv)
SCOPE=$(az storage account show --name $SA -g rg-az104-lab08 --query id -o tsv)

az role assignment create --assignee $USER \
  --role "Storage Blob Data Contributor" --scope $SCOPE

# Now list blobs with your identity — no key, no SAS
az storage blob list --account-name $SA --container-name docs --auth-mode login -o table
```

```powershell
$Scope = (Get-AzStorageAccount -ResourceGroupName rg-az104-lab08 -Name $Sa).Id
New-AzRoleAssignment -SignInName (Get-AzContext).Account.Id -RoleDefinitionName "Storage Blob Data Contributor" -Scope $Scope
```

## Clean up

```bash
az group delete --name rg-az104-lab08 --yes --no-wait
```

## References

- Storage account firewalls and virtual networks: https://learn.microsoft.com/azure/storage/common/storage-network-security
- Shared access signatures overview: https://learn.microsoft.com/azure/storage/common/storage-sas-overview
- Stored access policies: https://learn.microsoft.com/rest/api/storageservices/define-stored-access-policy
- Manage/rotate account access keys: https://learn.microsoft.com/azure/storage/common/storage-account-keys-manage

## What you learned

- List, rotate, and secure the two storage account access keys and understand why two keys exist.
- Generate account SAS and service (blob) SAS tokens scoped to least-privilege permissions and short expiries.
- Bind a SAS to a stored access policy so it can be revoked centrally.
- Lock a storage account behind its firewall using service endpoints on a VNet subnet plus allowed client IPs.
- Grant keyless, identity-based access using Azure RBAC data roles such as Storage Blob Data Contributor.
