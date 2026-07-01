# Lab 11 — Manage Data with Storage Explorer and AzCopy

Maps to AZ-104 objective **Implement and manage storage → Configure Azure Blob Storage / Configure access to storage** (manage data by using Azure Storage Explorer and AzCopy).

You will provision a storage account with two containers, then move data three ways: with the free **Azure Storage Explorer** desktop app (GUI), with **AzCopy** authenticated by a SAS token, and with AzCopy authenticated by Microsoft Entra login — including a server-to-server (account-to-account) copy that never touches your local disk.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com), plus the **Azure Storage Explorer** desktop app.

**Azure services used:** Azure Storage (Blob), Azure Storage Explorer (free desktop download), AzCopy (pre-installed in Cloud Shell), SAS, Azure RBAC.

---

## Step 1 — Create the resource group, account, and two containers

```bash
az group create --name rg-az104-lab11 --location eastus

SA=stlab11$RANDOM
echo "Storage account: $SA"   # globally unique lowercase name
az storage account create --name $SA --resource-group rg-az104-lab11 \
  --location eastus --sku Standard_LRS --kind StorageV2

az storage container create --account-name $SA --name source --auth-mode login
az storage container create --account-name $SA --name backup --auth-mode login
```

```powershell
New-AzResourceGroup -Name rg-az104-lab11 -Location eastus
$Sa = "stlab11$(Get-Random -Maximum 99999)"
New-AzStorageAccount -ResourceGroupName rg-az104-lab11 -Name $Sa -Location eastus -SkuName Standard_LRS -Kind StorageV2
```

## Step 2 — Grant yourself data-plane RBAC (for keyless AzCopy + login auth)

**Portal:** Account → **Access Control (IAM) → + Add → Add role assignment** → **Storage Blob Data Contributor** → your user → **Review + assign**.

```bash
USER=$(az ad signed-in-user show --query id -o tsv)
SCOPE=$(az storage account show --name $SA -g rg-az104-lab11 --query id -o tsv)
az role assignment create --assignee $USER --role "Storage Blob Data Contributor" --scope $SCOPE
```

## Step 3 — Install and connect Azure Storage Explorer (desktop)

Azure Storage Explorer is a **free desktop app** (Windows/macOS/Linux): https://azure.microsoft.com/products/storage/storage-explorer

**Desktop steps:**
1. Download and install Storage Explorer, then launch it.
2. Click the plug/**Connect** icon → **Subscription (Azure)** → sign in with your Azure account. Your subscriptions and storage accounts appear in the left tree.
3. Alternatively connect by **Storage account or service → Account name and key**, **Connection string**, or **Shared access signature (SAS) URL**.
4. Expand your account → **Blob Containers → source**.

## Step 4 — Upload and download with Storage Explorer (GUI)

**Desktop steps:**
1. Select the `source` container → **Upload → Upload Files** → pick a local file → **Upload**.
2. Select an uploaded blob → **Download** to save it locally.
3. Use **Copy / Paste** between the `source` and `backup` containers to move data with no scripting.
4. Right-click a blob → **Get Shared Access Signature** to generate a SAS URL from the GUI, or **Change access tier** to move it between Hot/Cool/Cold/Archive.

## Step 5 — Create test data and a SAS for AzCopy

AzCopy is pre-installed in Cloud Shell. It authenticates with either a **SAS token** appended to the URL or **`azcopy login`** (Entra).

```bash
mkdir -p ~/azcopydemo && echo "file one" > ~/azcopydemo/a.txt && echo "file two" > ~/azcopydemo/b.txt

KEY=$(az storage account keys list --account-name $SA -g rg-az104-lab11 --query "[0].value" -o tsv)
EXPIRY=$(date -u -d '2 hours' '+%Y-%m-%dT%H:%MZ' 2>/dev/null || date -u -v+2H '+%Y-%m-%dT%H:%MZ')

SAS=$(az storage container generate-sas --account-name $SA --account-key $KEY \
  --name source --permissions racwdl --expiry $EXPIRY --https-only -o tsv)
echo "SAS: $SAS"
```

## Step 6 — Upload a folder with AzCopy using a SAS

```bash
# Recursively upload the local folder into the source container
azcopy copy "$HOME/azcopydemo/*" \
  "https://$SA.blob.core.windows.net/source?$SAS" --recursive=true
# The ?$SAS query string authorizes the transfer; --recursive walks subfolders
```

List the results:

```bash
azcopy list "https://$SA.blob.core.windows.net/source?$SAS"
```

## Step 7 — Copy account-to-account (server-side) with AzCopy

AzCopy can copy directly between containers server-side — data never downloads to your machine. Here we copy `source` → `backup`.

```bash
BACKUPSAS=$(az storage container generate-sas --account-name $SA --account-key $KEY \
  --name backup --permissions racwl --expiry $EXPIRY --https-only -o tsv)

azcopy copy \
  "https://$SA.blob.core.windows.net/source?$SAS" \
  "https://$SA.blob.core.windows.net/backup?$BACKUPSAS" \
  --recursive=true
```

## Step 8 — Use AzCopy with Microsoft Entra login (keyless) and sync

Instead of SAS, authenticate AzCopy with your Entra identity (needs the RBAC role from Step 2).

```bash
# In Cloud Shell you are already signed in; on a local machine run:
# azcopy login   (or: azcopy login --identity  on an Azure VM with a managed identity)

# Download a container to a local folder using login auth (no SAS in the URL)
mkdir -p ~/restore
azcopy copy "https://$SA.blob.core.windows.net/backup" "$HOME/restore" --recursive=true

# 'azcopy sync' makes the destination match the source (incremental — only changed blobs)
azcopy sync "$HOME/azcopydemo" "https://$SA.blob.core.windows.net/source" --recursive=true
```

## Clean up

```bash
az group delete --name rg-az104-lab11 --yes --no-wait
```

Remove the Storage Explorer connection from the app's left tree if you connected via SAS/key.

## References

- Get started with Azure Storage Explorer: https://learn.microsoft.com/azure/storage/storage-explorer/vs-azure-tools-storage-manage-with-storage-explorer
- Get started with AzCopy: https://learn.microsoft.com/azure/storage/common/storage-use-azcopy-v10
- Transfer data with AzCopy and Blob storage: https://learn.microsoft.com/azure/storage/common/storage-use-azcopy-blobs-upload
- Authorize AzCopy with Microsoft Entra ID: https://learn.microsoft.com/azure/storage/common/storage-use-azcopy-authorize-azure-active-directory

## What you learned

- Install the free Azure Storage Explorer desktop app and connect via subscription, account key, connection string, or SAS URL.
- Upload, download, copy, and re-tier blobs entirely through the Storage Explorer GUI.
- Authorize AzCopy transfers with a scoped SAS token appended to the blob URL.
- Perform a server-side account-to-account copy with AzCopy so data never lands on your local disk.
- Authenticate AzCopy with Microsoft Entra login (keyless) and keep a container in sync with `azcopy sync`.
