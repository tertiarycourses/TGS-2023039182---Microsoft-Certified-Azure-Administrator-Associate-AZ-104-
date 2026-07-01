# Lab 26 — Azure Site Recovery: Replicate, Fail Over, and Interpret Health

Maps to AZ-104 objective **Monitor and maintain Azure resources → Implement disaster recovery** (configure Azure Site Recovery for Azure VMs; perform a failover to a secondary region; interpret replication health).

You will enable Azure-to-Azure replication for a VM into a secondary region, run a non-disruptive test failover, perform a planned/unplanned failover with re-protection and failback, and read replication and RPO health — the disaster-recovery workflow the exam tests. Azure Site Recovery is driven mainly from the Portal (the `az site-recovery` CLI is limited), so this lab leans on Portal click-paths while setting up the base resources with CLI.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Recovery Services vault (Site Recovery), Azure Site Recovery (Azure-to-Azure), a source Virtual Machine, source and target VNets, cache storage account.

---

## Step 1 — Create the resource group, source VM, and vault

**Portal:** Create **Resource group** `rg-az104-lab26` in a primary region (e.g. Southeast Asia). Create a VM `vm-az104-lab26` (Ubuntu, B1s) in it. Create a **Recovery Services vault** `rsv-asr-lab26` in the same region.

```bash
PRIMARY="southeastasia"
SECONDARY="eastasia"
RG="rg-az104-lab26"
VAULT="rsv-asr-lab26"

az group create --name "$RG" --location "$PRIMARY"

# source VM in the primary region (this is what we protect)
az vm create \
  --resource-group "$RG" \
  --name "vm-az104-lab26" \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --location "$PRIMARY" \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard

# Recovery Services vault that will host the Site Recovery configuration
az backup vault create \
  --resource-group "$RG" \
  --name "$VAULT" \
  --location "$PRIMARY"
```

```powershell
$RG="rg-az104-lab26"; $VAULT="rsv-asr-lab26"; $PRIMARY="southeastasia"
New-AzResourceGroup -Name $RG -Location $PRIMARY
New-AzRecoveryServicesVault -ResourceGroupName $RG -Name $VAULT -Location $PRIMARY
```

> For **Azure-to-Azure** replication, no agent install is required — ASR installs the Mobility service extension automatically when you enable replication.

## Step 2 — Enable replication for the VM

**Portal:** Open **rsv-asr-lab26 → Site Recovery → Enable Site Recovery → Azure virtual machines → Enable replication**. Source region = primary, select `vm-az104-lab26`. On the **Target** tab, ASR proposes a **target region** (choose East Asia), a **target resource group**, **target VNet**, **cache storage account**, **replication policy** (default: RPO 24h app-consistent snapshot, 24h crash-consistent, 3-day recovery-point retention), then **Enable replication**.

```bash
# ASR A2A is best configured in the portal. The base target resources can be pre-created via CLI:
az group create --name "rg-az104-lab26-dr" --location "$SECONDARY"

az network vnet create \
  --resource-group "rg-az104-lab26-dr" \
  --name "vnet-dr" \
  --location "$SECONDARY" \
  --address-prefix 10.20.0.0/16 \
  --subnet-name "default" --subnet-prefix 10.20.1.0/24

# cache storage account used by ASR to stage replication data in the source region
az storage account create \
  --resource-group "$RG" \
  --name "stasrcache$RANDOM" \
  --location "$PRIMARY" \
  --sku Standard_LRS
```

> The initial replication (seeding) can take 15–60+ minutes depending on disk size. Wait for the item to reach **Protected** before failing over.

## Step 3 — Configure/adjust the replication policy

**Portal:** **rsv-asr-lab26 → Site Recovery → Manage → Replication policies**. Open the auto-created policy (or **+ Create replication policy**) and set **App-consistent snapshot frequency** (e.g. every 4 hours) and **Recovery point retention** (e.g. 24 hours). Reassociate it with the replicated item if needed.

> Crash-consistent recovery points are taken every 5 minutes (RPO target). App-consistent points capture in-memory/app state at the configured frequency. Lower retention = fewer choices at failover.

## Step 4 — Interpret replication health

**Portal:** **rsv-asr-lab26 → Site Recovery → Replicated items → vm-az104-lab26**. Read: **Replication health** (Healthy/Warning/Critical), **Status** (Protected), **RPO** (current vs. target), and the list of available **recovery points**. Use **Site Recovery → Overview** for aggregate health and **Recovery Plans** readiness.

```bash
# CLI visibility into ASR is limited; use the portal Replicated items blade for health/RPO.
# You can confirm the vault and any ASR jobs exist:
az resource list --resource-group "$RG" \
  --resource-type "Microsoft.RecoveryServices/vaults" -o table
```

> Exam tip: **RPO** = how much data you could lose (gap since last recovery point). **RTO** = how long failover takes. A **Critical** replication health usually means missed RPO or an agent/heartbeat problem on the source.

## Step 5 — Run a test failover (non-disruptive)

**Portal:** Replicated item → **Test Failover**. Choose a recovery point (**Latest processed** recommended) and the **target VNet** (`vnet-dr`, isolated from production). ASR spins up a test VM in the secondary region without affecting the source or the replication direction. Validate the app, then **Cleanup test failover** and add a note.

> Always test-failover into an **isolated VNet** so the test VM can't clash with production. Test failover is the safe, exam-recommended way to validate DR readiness.

## Step 6 — Perform a failover to the secondary region

**Portal:** Replicated item → **Failover**. Pick a **recovery point** (Latest, Latest processed, Latest app-consistent, or Custom). Optionally tick **Shut down machine before beginning failover** (planned/graceful). Confirm — ASR creates the VM in the secondary region. Once validated, **Commit** the failover to finalize (this deletes the intermediate recovery points).

```bash
# after committing failover, the recovered VM exists in the DR resource group/region:
az vm list --resource-group "rg-az104-lab26-dr" -o table
```

> Failover types: use **Latest (lowest RPO)** for real disasters, **Latest app-consistent** when data integrity matters most, or a **Custom** recovery point. **Commit** only after you have verified the failed-over VM works.

## Step 7 — Re-protect and fail back to the primary region

**Portal:** After a failover, the item shows **Re-protect** — click it to start replicating the DR VM *back* to the primary region (reverses direction). Once re-protection is healthy, run **Failover** again (secondary → primary) and **Commit** to fail back. Then **Re-protect** once more to restore the original direction.

> The DR cycle is: **Enable replication → Failover → Commit → Re-protect → Failover (failback) → Commit → Re-protect**. Failback requires re-protection first; you cannot fail back directly.

## Step 8 — Build a Recovery Plan (orchestration)

**Portal:** **rsv-asr-lab26 → Site Recovery → Recovery Plans → + Recovery plan**. Name `rp-az104-lab26`, source/target regions, add `vm-az104-lab26`. Optionally add **groups**, **manual actions**, and **Azure Automation runbooks** to order multi-tier failovers (e.g. DB before app). You can test-failover or fail over the whole plan at once.

> Recovery Plans model real DR runbooks — group VMs by tier and inject scripts/manual steps so an entire application fails over in the correct order.

## Clean up

Site Recovery keeps replication state in the vault; you must **disable replication** on each item before removing the vault and resource groups.

**Portal:** Replicated item → **Disable Replication** (choose *remove replication settings*). Then **Site Recovery → clean any recovery plans**. Then delete the vault and both resource groups.

```bash
# 1) disable replication in the portal (Replicated items -> ... -> Disable Replication)
#    Then remove the vault. Recovery Services vaults with ASR/soft delete must be emptied first:
az backup vault backup-properties set --resource-group "$RG" --name "$VAULT" \
  --soft-delete-feature-state Disable 2>/dev/null || true
az backup vault delete --resource-group "$RG" --name "$VAULT" --yes 2>/dev/null || true

# 2) delete both the source and DR resource groups
az group delete --name rg-az104-lab26 --yes --no-wait
az group delete --name rg-az104-lab26-dr --yes --no-wait
```

> If vault deletion fails, replication is still enabled on an item, or the vault has soft-deleted backup items. Disable replication (and soft delete) and confirm the **Replicated items** list is empty before retrying.

## References

- About Azure Site Recovery — https://learn.microsoft.com/azure/site-recovery/site-recovery-overview
- Replicate Azure VMs to another region — https://learn.microsoft.com/azure/site-recovery/azure-to-azure-tutorial-enable-replication
- Run a test failover / failover — https://learn.microsoft.com/azure/site-recovery/site-recovery-test-failover-to-azure
- Reprotect and fail back Azure VMs — https://learn.microsoft.com/azure/site-recovery/azure-to-azure-tutorial-failback

## What you learned

- Enable **Azure-to-Azure Site Recovery** replication for a VM into a secondary region (no manual agent install).
- Configure a **replication policy** (app-consistent frequency and recovery-point retention) and understand crash- vs app-consistent points.
- Interpret **replication health**, **RPO**, and the difference between **RPO** and **RTO**.
- Run a **test failover** into an isolated VNet, then a real **failover** and **commit**.
- Complete the DR lifecycle with **re-protect** and **failback**, and orchestrate with a **Recovery Plan**.
- Correctly clean up by **disabling replication** before deleting the vault and resource groups.
