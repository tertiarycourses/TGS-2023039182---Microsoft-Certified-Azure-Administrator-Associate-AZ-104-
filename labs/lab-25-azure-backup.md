# Lab 25 — Azure Backup: Recovery Services Vault, Policies, Backup and Restore

Maps to AZ-104 objective **Monitor and maintain Azure resources → Implement backup and recovery** (create a Recovery Services vault; create and configure a backup policy; perform backup and restore with Azure Backup; configure and interpret backup reports and alerts).

You will create a Recovery Services vault, define a custom backup policy, protect a VM, run an on-demand backup, restore both a full VM and individual files, and set up backup reporting and alerts — the Azure Backup workflow the exam expects.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Recovery Services vault (and note the newer **Azure Backup vault**), Azure Backup, Backup policy, a Virtual Machine, Backup center / Backup reports, Azure Monitor alerts.

---

## Step 1 — Create the resource group and a Recovery Services vault

**Portal:** Search **Recovery Services vaults → + Create**. **Resource group** = `rg-az104-lab25`, **Name** = `rsv-az104-lab25`, **Region** = your region. On the vault's **Properties** you can later set **Storage replication** (GRS default) and **Soft delete** state.

```bash
LOCATION="southeastasia"
RG="rg-az104-lab25"
VAULT="rsv-az104-lab25"

az group create --name "$RG" --location "$LOCATION"

# Recovery Services vault protects Azure VMs, Azure Files, SQL-in-VM, on-prem via MARS, etc.
az backup vault create \
  --resource-group "$RG" \
  --name "$VAULT" \
  --location "$LOCATION"

# set backup storage redundancy to GRS (cross-region). Options: LocallyRedundant / GeoRedundant / ZoneRedundant
az backup vault backup-properties set \
  --resource-group "$RG" \
  --name "$VAULT" \
  --backup-storage-redundancy GeoRedundant
```

```powershell
$RG="rg-az104-lab25"; $VAULT="rsv-az104-lab25"; $LOCATION="southeastasia"
New-AzResourceGroup -Name $RG -Location $LOCATION
$vault = New-AzRecoveryServicesVault -ResourceGroupName $RG -Name $VAULT -Location $LOCATION
Set-AzRecoveryServicesBackupProperty -Vault $vault -BackupStorageRedundancy GeoRedundant
```

> There are two vault types on the exam: the **Recovery Services vault** (VMs, Azure Files, MARS, DPM/MABS) and the newer **Azure Backup vault** (blobs, disks, PostgreSQL, AKS, Azure Files with vaulted tier). This lab uses the Recovery Services vault for VM backup.

## Step 2 — Deploy a VM to protect

**Portal:** Create a small VM `vm-az104-lab25` (Ubuntu, B1s) in `rg-az104-lab25`.

```bash
az vm create \
  --resource-group "$RG" \
  --name "vm-az104-lab25" \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard
```

## Step 3 — Create and configure a custom backup policy

**Portal:** Open **rsv-az104-lab25 → Backup policies → + Add → Azure Virtual Machine**. Name `pol-daily-30d`, backup frequency **Daily 02:00**, retention **daily 30 days, weekly 12 weeks, monthly 12 months**. Save.

```bash
# start from the built-in DefaultPolicy JSON, edit retention, then create a custom policy
az backup policy show \
  --resource-group "$RG" --vault-name "$VAULT" \
  --name DefaultPolicy > policy.json

# edit policy.json to set schedule 02:00 and retention 30 daily / 12 weekly / 12 monthly, then:
az backup policy create \
  --resource-group "$RG" \
  --vault-name "$VAULT" \
  --name "pol-daily-30d" \
  --backup-management-type AzureIaasVM \
  --policy @policy.json
```

```powershell
$vault = Get-AzRecoveryServicesVault -Name $VAULT -ResourceGroupName $RG
Set-AzRecoveryServicesVaultContext -Vault $vault
$schedule = Get-AzRecoveryServicesBackupSchedulePolicyObject -WorkloadType AzureVM
$retention = Get-AzRecoveryServicesBackupRetentionPolicyObject -WorkloadType AzureVM
$retention.DailySchedule.DurationCountInDays = 30
New-AzRecoveryServicesBackupProtectionPolicy -Name "pol-daily-30d" -WorkloadType AzureVM `
  -RetentionPolicy $retention -SchedulePolicy $schedule
```

## Step 4 — Enable backup (protect the VM)

**Portal:** **rsv-az104-lab25 → Backup → Backup goal** = *Azure / Virtual machine*, select policy `pol-daily-30d`, add `vm-az104-lab25`, then **Enable backup**.

```bash
az backup protection enable-for-vm \
  --resource-group "$RG" \
  --vault-name "$VAULT" \
  --vm "vm-az104-lab25" \
  --policy-name "pol-daily-30d"

# confirm the item is now protected
az backup item list \
  --resource-group "$RG" --vault-name "$VAULT" \
  --query "[].{item:properties.friendlyName, health:properties.healthStatus}" -o table
```

```powershell
$vm = Get-AzVM -ResourceGroupName $RG -Name "vm-az104-lab25"
$pol = Get-AzRecoveryServicesBackupProtectionPolicy -Name "pol-daily-30d"
Enable-AzRecoveryServicesBackupProtection -ResourceGroupName $RG -Name $vm.Name -Policy $pol
```

## Step 5 — Run an on-demand backup

**Portal:** **rsv-az104-lab25 → Backup items → Azure Virtual Machine → vm-az104-lab25 → Backup now**. Choose a retention date and confirm. Watch the job under **Backup Jobs**.

```bash
# trigger an ad-hoc backup and keep the recovery point for 30 days
az backup protection backup-now \
  --resource-group "$RG" \
  --vault-name "$VAULT" \
  --container-name "vm-az104-lab25" \
  --item-name "vm-az104-lab25" \
  --backup-management-type AzureIaasVM \
  --retain-until $(date -u -v+30d +%d-%m-%Y 2>/dev/null || date -u -d '+30 days' +%d-%m-%Y)

# monitor the running/finished jobs
az backup job list --resource-group "$RG" --vault-name "$VAULT" \
  --query "[].{op:properties.operation, status:properties.status, start:properties.startTime}" -o table
```

## Step 6 — List recovery points

**Portal:** On the backup item's page, **Restore VM** shows selectable recovery points. Wait for the on-demand job to finish so at least one point exists.

```bash
az backup recoverypoint list \
  --resource-group "$RG" \
  --vault-name "$VAULT" \
  --container-name "vm-az104-lab25" \
  --item-name "vm-az104-lab25" \
  --backup-management-type AzureIaasVM \
  --query "[].{name:name, time:properties.recoveryPointTime}" -o table
```

## Step 7 — Restore individual files (Item-Level Recovery)

**Portal:** Backup item → **File recovery** → pick a recovery point → **Download script**. Run the script on any machine to mount the recovery point as a drive, copy files, then **Unmount disks**.

```bash
RP=$(az backup recoverypoint list -g "$RG" --vault-name "$VAULT" \
  --container-name "vm-az104-lab25" --item-name "vm-az104-lab25" \
  --backup-management-type AzureIaasVM --query "[0].name" -o tsv)

# generate the ILR mount script + password (mount, copy files, then unmount)
az backup restore files mount-rp \
  --resource-group "$RG" --vault-name "$VAULT" \
  --container-name "vm-az104-lab25" --item-name "vm-az104-lab25" \
  --rp-name "$RP"

# when finished copying, unmount
az backup restore files unmount-rp \
  --resource-group "$RG" --vault-name "$VAULT" \
  --container-name "vm-az104-lab25" --item-name "vm-az104-lab25" \
  --rp-name "$RP"
```

## Step 8 — Restore the full VM (to disks / new VM)

**Portal:** Backup item → **Restore VM** → select a recovery point → **Create new** VM or **Replace existing**. For a safe lab restore, choose **Create new** into a staging storage account and a different VM name.

```bash
STORAGE="stlab25$RANDOM"
az storage account create -g "$RG" -n "$STORAGE" --sku Standard_LRS --location "$LOCATION"

# restore the recovery point to managed disks in a staging account (then build a new VM from them)
az backup restore restore-disks \
  --resource-group "$RG" \
  --vault-name "$VAULT" \
  --container-name "vm-az104-lab25" \
  --item-name "vm-az104-lab25" \
  --rp-name "$RP" \
  --storage-account "$STORAGE" \
  --target-resource-group "$RG"

# track the restore job
az backup job list --resource-group "$RG" --vault-name "$VAULT" -o table
```

> Restore options: **Create new VM**, **Replace existing** (uses a staging account for disks), **Restore disks**, or **Cross Region Restore** (only if the vault is GRS with CRR enabled).

## Step 9 — Configure and interpret backup reports and alerts

**Portal:** Open **Backup center → Backup reports** (a Log Analytics-backed workbook — configure the vault's diagnostic settings to a workspace first). For alerting, use **Backup center → Alerts** (built-in Azure Monitor alerts for backup/restore failures) and attach an action group.

```bash
# 1) send vault diagnostics to a workspace so Backup Reports has data
WS_ID=$(az monitor log-analytics workspace create -g "$RG" -n "law-az104-lab25" \
  --location "$LOCATION" --query id -o tsv)
VAULT_ID=$(az backup vault show -g "$RG" -n "$VAULT" --query id -o tsv)

az monitor diagnostic-settings create \
  --name "vault-diag" \
  --resource "$VAULT_ID" \
  --workspace "$WS_ID" \
  --logs '[{"category":"CoreAzureBackup","enabled":true},{"category":"AddonAzureBackupJobs","enabled":true},{"category":"AddonAzureBackupPolicy","enabled":true}]'

# 2) view built-in backup alerts (Azure Monitor alerts for backup) — check the current jobs' status
az backup job list -g "$RG" --vault-name "$VAULT" \
  --query "[?properties.status=='Failed'].{op:properties.operation, err:properties.status}" -o table
```

> Azure Backup surfaces alerts two ways: **built-in Azure Monitor alerts** (recommended; failure alerts routed to action groups) and **Backup reports** (a workbook aggregating job success, storage consumption, and policy adherence across vaults).

## Clean up

A Recovery Services vault **cannot be deleted while it protects items or holds recovery points** — you must stop protection and delete backup data first, and (if enabled) work around soft delete.

**Portal:** Vault → **Properties → Security Settings** → disable **Soft delete** if you need immediate deletion. Then **Backup items → Stop backup → Delete backup data** for each item. Finally delete the vault, then the RG.

```bash
# 1) stop protection and delete backup data for the VM item
az backup protection disable \
  --resource-group "$RG" --vault-name "$VAULT" \
  --container-name "vm-az104-lab25" --item-name "vm-az104-lab25" \
  --backup-management-type AzureIaasVM \
  --delete-backup-data true --yes

# 2) disable soft delete so already-deleted items aren't retained for 14 days
az backup vault backup-properties set \
  --resource-group "$RG" --name "$VAULT" \
  --soft-delete-feature-state Disable

# 3) (if any items are in soft-deleted state) undelete then re-delete, or wait, then:
az backup vault delete --resource-group "$RG" --name "$VAULT" --yes 2>/dev/null || true

# 4) finally remove everything else
az group delete --name rg-az104-lab25 --yes --no-wait
```

> If the vault delete fails, it still has protected/soft-deleted items or unregistered containers. Confirm `az backup item list` returns empty and soft delete is off before retrying.

## References

- Recovery Services vault overview — https://learn.microsoft.com/azure/backup/backup-azure-recovery-services-vault-overview
- Back up an Azure VM — https://learn.microsoft.com/azure/backup/backup-azure-arm-vms-prepare
- Restore Azure VMs — https://learn.microsoft.com/azure/backup/backup-azure-arm-restore-vms
- Backup reports and alerts — https://learn.microsoft.com/azure/backup/backup-azure-monitoring-use-azuremonitor

## What you learned

- Create a **Recovery Services vault**, set its storage redundancy, and distinguish it from the newer **Azure Backup vault**.
- Author a **custom backup policy** (schedule + daily/weekly/monthly retention) and apply it to a VM.
- Enable protection, run an **on-demand backup**, and enumerate **recovery points**.
- Perform both **item-level file recovery** and a **full VM/disk restore**, and know the four restore options.
- Configure vault diagnostics for **Backup reports** and enable built-in **backup alerts** through an action group.
- Correctly tear down a vault by stopping protection, deleting backup data, and disabling **soft delete**.
