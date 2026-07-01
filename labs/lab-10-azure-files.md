# Lab 10 — Configure Azure Files and Identity-Based Access

Maps to AZ-104 objective **Implement and manage storage → Configure Azure Files** (create and configure a file share in Azure Files; configure snapshots and soft delete for Azure Files; configure identity-based access for Azure Files).

You will create an SMB file share, mount it, take and restore share snapshots, enable soft delete, and configure identity-based (Microsoft Entra / Active Directory) authentication so users mount the share with their own credentials instead of the storage account key.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Azure Files (SMB shares), share snapshots, file-share soft delete, identity-based authentication (Microsoft Entra Domain Services / AD DS / Entra Kerberos), Azure RBAC share roles.

---

## Step 1 — Create the resource group and storage account

Azure Files uses a **general-purpose v2** (or FileStorage) account. Standard shares use HDD; premium uses SSD (`FileStorage` + `Premium_LRS`).

```bash
az group create --name rg-az104-lab10 --location eastus

SA=stlab10$RANDOM
echo "Storage account: $SA"   # globally unique lowercase name
az storage account create --name $SA --resource-group rg-az104-lab10 \
  --location eastus --sku Standard_LRS --kind StorageV2
```

```powershell
New-AzResourceGroup -Name rg-az104-lab10 -Location eastus
$Sa = "stlab10$(Get-Random -Maximum 99999)"
New-AzStorageAccount -ResourceGroupName rg-az104-lab10 -Name $Sa -Location eastus -SkuName Standard_LRS -Kind StorageV2
```

## Step 2 — Create a file share

Use the management-plane `share-rm` commands (recommended) so quota and tier are set at the ARM layer.

**Portal:** Account → **Data storage → File shares → + File share** → name `projects`, tier *Transaction optimized*, set a quota → **Create**.

```bash
az storage share-rm create \
  --resource-group rg-az104-lab10 --storage-account $SA \
  --name projects --quota 100 --access-tier TransactionOptimized
```

```powershell
New-AzRmStorageShare -ResourceGroupName rg-az104-lab10 -StorageAccountName $Sa -Name projects -QuotaGiB 100
```

## Step 3 — Upload a file and view the mount command

**Portal:** File share `projects` → **Upload** to add files. Click **Connect** to view a ready-made script for Windows/Linux/macOS.

```bash
KEY=$(az storage account keys list --account-name $SA -g rg-az104-lab10 --query "[0].value" -o tsv)
echo "quarterly plan" > plan.txt
az storage file upload --account-name $SA --account-key $KEY \
  --share-name projects --source plan.txt

az storage file list --account-name $SA --account-key $KEY --share-name projects -o table
```

To mount with the account key (Linux example from the **Connect** blade):

```bash
# sudo mount -t cifs //$SA.file.core.windows.net/projects /mnt/projects \
#   -o vers=3.0,username=$SA,password=$KEY,dir_mode=0777,file_mode=0777,serverino
```

## Step 4 — Enable share soft delete

Soft delete lets you recover a deleted **file share** within a retention window (share-level; set on the file service).

**Portal:** Account → **Data protection** → under *Recovery* tick **Enable soft delete for file shares** and set days (e.g. 7) → **Save**.

```bash
az storage account file-service-properties update \
  --account-name $SA --resource-group rg-az104-lab10 \
  --enable-delete-retention true --delete-retention-days 7

# Verify
az storage account file-service-properties show \
  --account-name $SA --resource-group rg-az104-lab10 \
  --query "shareDeleteRetentionPolicy" -o json
```

## Step 5 — Take a share snapshot

Snapshots are read-only point-in-time copies of the entire share, useful for backup/versioning.

**Portal:** File share → **Snapshots → + Add snapshot**.

```bash
# Create a snapshot and capture its timestamp
SNAP=$(az storage share snapshot --account-name $SA --account-key $KEY \
  --name projects --query snapshot -o tsv)
echo "Snapshot: $SNAP"

# List snapshots
az storage share list --account-name $SA --account-key $KEY \
  --include-snapshots --query "[?name=='projects'].{name:name, snapshot:snapshot}" -o table
```

## Step 6 — Restore a file from a snapshot

**Portal:** File share → **Snapshots** → open a snapshot → select a file → **Restore** (or download).

```bash
# Browse files inside the snapshot
az storage file list --account-name $SA --account-key $KEY \
  --share-name projects --snapshot "$SNAP" -o table

# Copy a file out of the snapshot back into the live share
az storage file copy start --account-name $SA --account-key $KEY \
  --source-account-name $SA --source-account-key $KEY \
  --source-share projects --source-path plan.txt --source-snapshot "$SNAP" \
  --destination-share projects --destination-path plan-restored.txt
```

## Step 7 — Configure identity-based access (authentication)

Azure Files supports three identity sources for SMB: **on-prem AD DS**, **Microsoft Entra Domain Services**, and **Microsoft Entra Kerberos** (for hybrid/Entra-joined clients). This replaces key-based mounting with the user's own identity.

**Portal:** Account → **Data storage → File shares → Active Directory** (or **Security + networking → Identity**) → choose **Microsoft Entra Domain Services**, **On-premises AD DS**, or **Microsoft Entra Kerberos** → **Set up / Enable**.

```bash
# Example: enable Microsoft Entra Domain Services (Azure AD DS) authentication for the account
az storage account update --name $SA --resource-group rg-az104-lab10 \
  --enable-files-aadds true
# For on-prem AD DS you instead join the account with the AzFilesHybrid PowerShell module,
# which sets --enable-files-adds and registers a computer/service account in your domain.
```

## Step 8 — Assign share-level RBAC roles for identity-based access

After enabling an identity source, users still need a **share-level RBAC role** plus NTFS permissions.

**Portal:** File share → **Access Control (IAM) → + Add role assignment** → e.g. **Storage File Data SMB Share Contributor** → assign to a user/group.

```bash
USER=$(az ad signed-in-user show --query id -o tsv)
SHARESCOPE=$(az storage account show --name $SA -g rg-az104-lab10 --query id -o tsv)/fileServices/default/fileshares/projects

az role assignment create --assignee $USER \
  --role "Storage File Data SMB Share Contributor" \
  --scope "$SHARESCOPE"
# Elevated (Owner-like) NTFS admin role is "Storage File Data SMB Share Elevated Contributor"
```

## Clean up

```bash
az group delete --name rg-az104-lab10 --yes --no-wait
```

## References

- Azure Files planning guide: https://learn.microsoft.com/azure/storage/files/storage-files-planning
- Share snapshots: https://learn.microsoft.com/azure/storage/files/storage-snapshots-files
- Enable soft delete for Azure file shares: https://learn.microsoft.com/azure/storage/files/storage-files-enable-soft-delete
- Overview of identity-based authentication for Azure Files: https://learn.microsoft.com/azure/storage/files/storage-files-active-directory-overview

## What you learned

- Create and quota an SMB file share and view its Portal-generated mount command.
- Enable file-share soft delete to recover accidentally deleted shares.
- Take share snapshots and restore individual files from a point-in-time snapshot.
- Choose among AD DS, Entra Domain Services, and Entra Kerberos identity sources for Azure Files.
- Grant share-level access with RBAC roles such as Storage File Data SMB Share Contributor for keyless, identity-based mounting.
