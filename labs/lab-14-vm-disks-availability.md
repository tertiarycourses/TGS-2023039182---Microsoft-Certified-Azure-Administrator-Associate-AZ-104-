# Lab 14 — VM Sizes, Disks, and Availability (Zones & Sets)

Maps to AZ-104 objective **Deploy and manage Azure compute resources → Manage virtual machines** (manage VM sizes, manage VM disks — attach/expand a data disk — and deploy VMs to availability zones and availability sets).

In this lab you will resize a VM, attach and expand a managed data disk, and deploy VMs into an availability zone and into an availability set for high availability.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Azure Virtual Machines, Managed Disks, Availability Zones, Availability Sets.

> Cost warning: **running VMs and premium disks cost money.** Use `Standard_B1s`, `deallocate` when idle, and delete everything in **Clean up**.

---

## Step 1 — Create the resource group and a base VM

**Portal:** **Resource groups → + Create** → `rg-az104-lab14` / `East US`. Then **Virtual machines → + Create** → `vm-lab14` → Ubuntu 22.04 → `Standard_B1s`.

```bash
# Azure CLI
az group create --name rg-az104-lab14 --location eastus

az vm create \
  --resource-group rg-az104-lab14 \
  --name vm-lab14 \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard
```

```powershell
New-AzResourceGroup -Name rg-az104-lab14 -Location eastus
# New-AzVM -ResourceGroupName rg-az104-lab14 -Name vm-lab14 -Image Ubuntu2204 -Size Standard_B1s
```

---

## Step 2 — List and manage VM sizes

**Portal:** Open **vm-lab14 → Size** → review available sizes and their vCPU/RAM/cost.

```bash
# List sizes available in the region
az vm list-sizes --location eastus --output table

# List sizes this specific VM can resize to (same hardware cluster)
az vm list-vm-resize-options --resource-group rg-az104-lab14 --name vm-lab14 --output table
```

```powershell
Get-AzVMSize -Location eastus | Format-Table
Get-AzVMSize -ResourceGroupName rg-az104-lab14 -VMName vm-lab14 | Format-Table
```

---

## Step 3 — Resize the VM

**Portal:** **vm-lab14 → Size** → choose `Standard_B2s` → **Resize**. (If the size isn't on the current cluster, the VM must be deallocated first.)

```bash
# Azure CLI — az vm resize
az vm resize --resource-group rg-az104-lab14 --name vm-lab14 --size Standard_B2s

# If the target size needs different hardware:
az vm deallocate -g rg-az104-lab14 -n vm-lab14
az vm resize -g rg-az104-lab14 -n vm-lab14 --size Standard_B2s
az vm start -g rg-az104-lab14 -n vm-lab14
```

```powershell
$vm = Get-AzVM -ResourceGroupName rg-az104-lab14 -VMName vm-lab14
$vm.HardwareProfile.VmSize = "Standard_B2s"
Update-AzVM -ResourceGroupName rg-az104-lab14 -VM $vm
```

---

## Step 4 — Create and attach a managed data disk

**Portal:** **vm-lab14 → Disks → + Create and attach a new disk** → Size `32 GiB`, `Standard SSD` → **Save**.

```bash
# Create an empty managed disk
az disk create \
  --resource-group rg-az104-lab14 \
  --name datadisk-lab14 \
  --size-gb 32 \
  --sku StandardSSD_LRS

# Attach it to the VM
az vm disk attach \
  --resource-group rg-az104-lab14 \
  --vm-name vm-lab14 \
  --name datadisk-lab14

# Alternatively create + attach in one step:
az vm disk attach -g rg-az104-lab14 --vm-name vm-lab14 --name datadisk2 --new --size-gb 16
```

```powershell
$diskConfig = New-AzDiskConfig -Location eastus -DiskSizeGB 32 -SkuName StandardSSD_LRS -CreateOption Empty
$disk = New-AzDisk -ResourceGroupName rg-az104-lab14 -DiskName datadisk-lab14 -Disk $diskConfig
$vm = Get-AzVM -ResourceGroupName rg-az104-lab14 -Name vm-lab14
Add-AzVMDataDisk -VM $vm -Name datadisk-lab14 -ManagedDiskId $disk.Id -Lun 0 -CreateOption Attach
Update-AzVM -ResourceGroupName rg-az104-lab14 -VM $vm
```

---

## Step 5 — Expand the data disk

Disks can only grow, never shrink. The VM should be **deallocated** for OS/data disk expansion in most cases.

**Portal:** **vm-lab14 → Disks →** click the data disk → **Size + performance** → set `64 GiB` → **Save**.

```bash
# Deallocate, then expand
az vm deallocate -g rg-az104-lab14 -n vm-lab14
az disk update --resource-group rg-az104-lab14 --name datadisk-lab14 --size-gb 64
az vm start -g rg-az104-lab14 -n vm-lab14
# Inside the guest OS you must then grow the partition/filesystem (e.g. growpart + resize2fs).
```

```powershell
$disk = Get-AzDisk -ResourceGroupName rg-az104-lab14 -DiskName datadisk-lab14
$disk.DiskSizeGB = 64
Update-AzDisk -ResourceGroupName rg-az104-lab14 -DiskName datadisk-lab14 -Disk $disk
```

# Expanding the managed disk only grows the backing storage; you still extend the volume inside the OS.

---

## Step 6 — Deploy a VM into an availability zone

Availability zones protect against datacenter-level failures within a region.

**Portal:** **+ Create VM** → **Availability options = Availability zone** → pick **Zone 1** → deploy.

```bash
# Deploy directly into zone 1 with --zone
az vm create \
  --resource-group rg-az104-lab14 \
  --name vm-zone1 \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --zone 1 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard
```

```powershell
# New-AzVM -ResourceGroupName rg-az104-lab14 -Name vm-zone1 -Zone 1 -Size Standard_B1s -Image Ubuntu2204
```

# Zonal VMs use zone-redundant/zonal public IPs and Standard SKU load balancers for resilience.

---

## Step 7 — Create an availability set and deploy VMs into it

Availability sets protect against rack/host failures within a single datacenter using fault and update domains.

**Portal:** Search **Availability sets → + Create** → `avset-lab14`, RG `rg-az104-lab14`, Fault domains `2`, Update domains `5`, **Use managed disks = Aligned**. Then create a VM with **Availability options = Availability set → avset-lab14**.

```bash
# Create the availability set (aligned = managed disks)
az vm availability-set create \
  --resource-group rg-az104-lab14 \
  --name avset-lab14 \
  --platform-fault-domain-count 2 \
  --platform-update-domain-count 5

# Deploy two VMs into the set
az vm create -g rg-az104-lab14 -n vm-set-1 --availability-set avset-lab14 \
  --image Ubuntu2204 --size Standard_B1s --admin-username azureuser --generate-ssh-keys
az vm create -g rg-az104-lab14 -n vm-set-2 --availability-set avset-lab14 \
  --image Ubuntu2204 --size Standard_B1s --admin-username azureuser --generate-ssh-keys
```

```powershell
New-AzAvailabilitySet -ResourceGroupName rg-az104-lab14 -Name avset-lab14 `
  -Location eastus -PlatformFaultDomainCount 2 -PlatformUpdateDomainCount 5 -Sku Aligned
```

# You cannot combine an availability set and an availability zone for the same VM — choose one.

---

## Clean up

```bash
az group delete --name rg-az104-lab14 --yes --no-wait
```

## References
- Resize a virtual machine: https://learn.microsoft.com/azure/virtual-machines/resize-vm
- Attach a managed data disk to a Linux VM with Azure CLI: https://learn.microsoft.com/azure/virtual-machines/linux/add-disk
- Expand virtual hard disks on a VM: https://learn.microsoft.com/azure/virtual-machines/linux/expand-disks
- Availability options for VMs — zones and availability sets: https://learn.microsoft.com/azure/virtual-machines/availability

## What you learned
- How to list VM sizes and resize a VM with `az vm resize`.
- How to create a managed data disk with `az disk create` and attach it with `az vm disk attach`.
- How to expand a data disk with `az disk update` (grow only; then extend inside the OS).
- How to deploy a VM into an availability zone with `--zone`.
- How to create an availability set (fault/update domains) and place VMs in it.
- The difference between availability zones and availability sets, and that they are mutually exclusive per VM.
