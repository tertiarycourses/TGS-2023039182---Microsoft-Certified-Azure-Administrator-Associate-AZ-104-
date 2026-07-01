# Lab 13 — Create a Virtual Machine, Encryption at Host, and Moving VMs

Maps to AZ-104 objective **Deploy and manage Azure compute resources → Create and configure virtual machines** (create a VM, configure encryption at host, move a VM to another resource group / subscription / region).

In this lab you will deploy a Linux VM, enable **encryption at host**, then move the VM between resource groups and to another region using Azure Resource Mover.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Azure Virtual Machines, Managed Disks, Azure Resource Mover, Azure Resource Manager.

> Cost warning: **VMs incur charges while running.** Use `Standard_B1s` (burstable, cheap), **deallocate** when idle, and delete everything in **Clean up**.

---

## Step 1 — Create resource groups

**Portal:** **Resource groups → + Create** → Name `rg-az104-lab13` → Region `East US` → **Create**. Repeat for `rg-az104-lab13-dest`.

```bash
# Azure CLI
az group create --name rg-az104-lab13 --location eastus
az group create --name rg-az104-lab13-dest --location eastus
```

```powershell
New-AzResourceGroup -Name rg-az104-lab13 -Location eastus
New-AzResourceGroup -Name rg-az104-lab13-dest -Location eastus
```

---

## Step 2 — Register the encryption-at-host feature

Encryption at host must be enabled once per subscription before it can be used.

```bash
az feature register --namespace Microsoft.Compute --name EncryptionAtHost
# Wait until state is "Registered", then re-register the provider
az feature show --namespace Microsoft.Compute --name EncryptionAtHost --query properties.state -o tsv
az provider register --namespace Microsoft.Compute
```

# Registration can take several minutes. Encryption at host encrypts VM temp disks and OS/data disk caches on the host itself.

---

## Step 3 — Create a virtual machine with encryption at host

**Portal:** **Virtual machines → + Create → Azure virtual machine** → RG `rg-az104-lab13` → Name `vm-lab13` → Image `Ubuntu Server 22.04 LTS` → Size `Standard_B1s` → Authentication `SSH public key` → On the **Disks** tab, enable **Encryption at host** → **Review + create**.

```bash
# Azure CLI — --encryption-at-host enables host-based encryption at create time
az vm create \
  --resource-group rg-az104-lab13 \
  --name vm-lab13 \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --encryption-at-host \
  --public-ip-sku Standard
```

```powershell
# Azure PowerShell — set EncryptionAtHost on the VM config
$cred = Get-Credential -UserName azureuser
$vmConfig = New-AzVMConfig -VMName vm-lab13 -VMSize Standard_B1s -EncryptionAtHost $true
# ...add OS, image, NIC to $vmConfig as usual...
New-AzVM -ResourceGroupName rg-az104-lab13 -Location eastus -VM $vmConfig
```

---

## Step 4 — Verify encryption at host on an existing VM

**Portal:** Open **vm-lab13 → Disks → Additional settings** → confirm **Encryption at host** is **Enabled**.

```bash
# Enable encryption at host on an existing VM (must be deallocated first)
az vm deallocate --resource-group rg-az104-lab13 --name vm-lab13
az vm update --resource-group rg-az104-lab13 --name vm-lab13 \
  --set securityProfile.encryptionAtHost=true
az vm start --resource-group rg-az104-lab13 --name vm-lab13

# Confirm
az vm show -g rg-az104-lab13 -n vm-lab13 --query securityProfile.encryptionAtHost
```

---

## Step 5 — Move a VM to another resource group

**Portal:** Open **vm-lab13 → Overview → Move → Move to another resource group** → tick the VM and its dependent resources (disk, NIC, IP) → choose `rg-az104-lab13-dest` → confirm → **OK**.

```bash
# Get resource IDs of the VM and its dependencies, then move them together
VMID=$(az vm show -g rg-az104-lab13 -n vm-lab13 --query id -o tsv)
az resource move --destination-group rg-az104-lab13-dest --ids $VMID
# NOTE: In practice you must move the VM AND its disk, NIC, and public IP in one call.
```

```powershell
$vm = Get-AzVM -ResourceGroupName rg-az104-lab13 -Name vm-lab13
Move-AzResource -DestinationResourceGroupName rg-az104-lab13-dest -ResourceId $vm.Id
```

# Moving between resource groups keeps the same region and subscription. All dependent resources must move together.

---

## Step 6 — Move a VM to another subscription

**Portal:** **vm-lab13 → Move → Move to another subscription** → pick target subscription and RG → validate → **OK**.

```bash
# Requires a second subscription you have access to
TARGET_SUB="<destination-subscription-id>"
az resource move \
  --destination-group rg-az104-lab13-dest \
  --destination-subscription-id $TARGET_SUB \
  --ids $VMID
```

# Cross-subscription moves require the destination RG to exist in the target subscription.

---

## Step 7 — Move a VM to another region with Azure Resource Mover

**Portal:** Search **Azure Resource Mover** → **+ Move resources across regions** → Source region `East US`, target `West US 2` → select **vm-lab13** and dependencies → **Add** → **Prepare** → **Initiate move** → after validation, **Commit move**.

```bash
# High-level CLI flow (Resource Mover)
# 1. Create a move collection in the target region
az resource-mover move-collection create \
  --name mc-lab13 --resource-group rg-az104-lab13 \
  --source-region eastus --target-region westus2 --location eastus2euap --identity type=SystemAssigned

# 2. Add the VM, then run Resolve-dependencies, Prepare, Initiate, Commit stages
az resource-mover move-resource add \
  --move-collection-name mc-lab13 --resource-group rg-az104-lab13 \
  --name vm-lab13-move --source-id $VMID
```

# Cross-region move copies resources into the target region; validate, prepare, initiate, then commit or discard.

---

## Clean up

```bash
az group delete --name rg-az104-lab13 --yes --no-wait
az group delete --name rg-az104-lab13-dest --yes --no-wait
```

## References
- Quickstart: Create a Linux VM with Azure CLI: https://learn.microsoft.com/azure/virtual-machines/linux/quick-create-cli
- Enable end-to-end encryption using encryption at host: https://learn.microsoft.com/azure/virtual-machines/disks-enable-host-based-encryption-portal
- Move resources to a new resource group or subscription: https://learn.microsoft.com/azure/azure-resource-manager/management/move-resource-group-and-subscription
- Move Azure VMs across regions with Azure Resource Mover: https://learn.microsoft.com/azure/resource-mover/tutorial-move-region-virtual-machines

## What you learned
- How to register and use **encryption at host** on both new and existing VMs.
- How to create a Linux VM with `az vm create --encryption-at-host`.
- How to move a VM to a different resource group with `az resource move` (dependencies must move together).
- How to move a VM to another subscription.
- How to move a VM to another region with Azure Resource Mover (prepare → initiate → commit).
- Why you should deallocate VMs to control cost.
