# Lab 15 — Deploy and Configure a Virtual Machine Scale Set (VMSS) with Autoscale

Maps to AZ-104 objective **Deploy and manage Azure compute resources → Deploy and configure a Virtual Machine Scale Set** (create a scale set, configure instances, and define autoscale rules based on metrics).

In this lab you will create a Virtual Machine Scale Set behind a load balancer, then attach autoscale rules that add and remove instances based on CPU usage.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Azure Virtual Machine Scale Sets, Azure Load Balancer, Azure Monitor Autoscale.

> Cost warning: **each instance is a billed VM.** Keep the size small (`Standard_B1s`) and low instance counts, and delete everything in **Clean up**.

---

## Step 1 — Create the resource group

**Portal:** **Resource groups → + Create** → `rg-az104-lab15` / `East US`.

```bash
az group create --name rg-az104-lab15 --location eastus
```

```powershell
New-AzResourceGroup -Name rg-az104-lab15 -Location eastus
```

---

## Step 2 — Create the Virtual Machine Scale Set

**Portal:** Search **Virtual machine scale sets → + Create** → RG `rg-az104-lab15` → Name `vmss-lab15` → Image `Ubuntu Server 22.04 LTS` → Size `Standard_B1s` → **Orchestration mode = Flexible** → Instance count `2` → **Load balancing = Azure load balancer** → **Review + create**.

```bash
# Azure CLI — Flexible orchestration, 2 instances, behind a new load balancer
az vmss create \
  --resource-group rg-az104-lab15 \
  --name vmss-lab15 \
  --orchestration-mode Flexible \
  --image Ubuntu2204 \
  --vm-sku Standard_B1s \
  --instance-count 2 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --load-balancer vmss-lb \
  --upgrade-policy-mode automatic
```

```powershell
# New-AzVmss -ResourceGroupName rg-az104-lab15 -VMScaleSetName vmss-lab15 `
#   -InstanceCount 2 -SkuName Standard_B1s -ImageName Ubuntu2204 -OrchestrationMode Flexible
```

# Flexible orchestration is the current default; instances behave like regular VMs and can span zones.

---

## Step 3 — View and manually scale instances

**Portal:** **vmss-lab15 → Instances** to see running instances; **vmss-lab15 → Scaling → Manual scale** to set a fixed count.

```bash
# List instances
az vmss list-instances --resource-group rg-az104-lab15 --name vmss-lab15 --output table

# Manually scale to 3 instances
az vmss scale --resource-group rg-az104-lab15 --name vmss-lab15 --new-capacity 3
```

```powershell
Get-AzVmssVM -ResourceGroupName rg-az104-lab15 -VMScaleSetName vmss-lab15
Update-AzVmss -ResourceGroupName rg-az104-lab15 -Name vmss-lab15 -InstanceCount 3 | Out-Null
```

---

## Step 4 — Create an autoscale profile

**Portal:** **vmss-lab15 → Scaling → Custom autoscale** → Name `autoscale-lab15` → Min `2`, Max `10`, Default `2`.

```bash
# Create the autoscale setting targeting the scale set
az monitor autoscale create \
  --resource-group rg-az104-lab15 \
  --resource vmss-lab15 \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name autoscale-lab15 \
  --min-count 2 \
  --max-count 10 \
  --count 2
```

```powershell
# New-AzAutoscaleSetting -Name autoscale-lab15 -ResourceGroupName rg-az104-lab15 `
#   -TargetResourceId <vmssId> -AutoscaleProfile <profile>
```

---

## Step 5 — Add a scale-out rule (CPU high)

Add one instance when average CPU exceeds 70% for 5 minutes.

**Portal:** In the autoscale profile → **+ Add a rule** → Metric `Percentage CPU`, Operator `Greater than`, Threshold `70`, Duration `5 min` → Action `Increase count by 1`, Cool down `5 min`.

```bash
az monitor autoscale rule create \
  --resource-group rg-az104-lab15 \
  --autoscale-name autoscale-lab15 \
  --condition "Percentage CPU > 70 avg 5m" \
  --scale out 1
```

# "Scale out" adds instances; the cool-down prevents rapid flapping.

---

## Step 6 — Add a scale-in rule (CPU low)

Remove one instance when average CPU falls below 30% for 5 minutes.

**Portal:** **+ Add a rule** → Metric `Percentage CPU`, Operator `Less than`, Threshold `30`, Duration `5 min` → Action `Decrease count by 1`.

```bash
az monitor autoscale rule create \
  --resource-group rg-az104-lab15 \
  --autoscale-name autoscale-lab15 \
  --condition "Percentage CPU < 30 avg 5m" \
  --scale in 1
```

# Always pair a scale-out rule with a scale-in rule so the set shrinks when load drops.

---

## Step 7 — Review autoscale configuration

**Portal:** **vmss-lab15 → Scaling → Run history / JSON** to confirm both rules and the min/max bounds.

```bash
az monitor autoscale show --resource-group rg-az104-lab15 --name autoscale-lab15 --output json
az monitor autoscale rule list --resource-group rg-az104-lab15 --autoscale-name autoscale-lab15 --output table
```

---

## Step 8 — (Optional) Update the VMSS instance size / image

**Portal:** **vmss-lab15 → Size** to change SKU; **Instances → Upgrade** to apply the latest model.

```bash
# Change the SKU for future instances
az vmss update --resource-group rg-az104-lab15 --name vmss-lab15 --set sku.name=Standard_B2s
# Roll the change to existing instances (manual upgrade policy)
az vmss update-instances --resource-group rg-az104-lab15 --name vmss-lab15 --instance-ids '*'
```

---

## Clean up

```bash
az group delete --name rg-az104-lab15 --yes --no-wait
```

## References
- Quickstart: Create a VM scale set with Azure CLI: https://learn.microsoft.com/azure/virtual-machine-scale-sets/quick-create-cli
- Overview of autoscale with Azure VM scale sets: https://learn.microsoft.com/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-autoscale-overview
- Create autoscale settings with the CLI: https://learn.microsoft.com/azure/azure-monitor/autoscale/autoscale-using-powershell
- Orchestration modes for VM scale sets: https://learn.microsoft.com/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-orchestration-modes

## What you learned
- How to create a VMSS in Flexible orchestration mode behind a load balancer with `az vmss create`.
- How to view and manually scale instances with `az vmss scale`.
- How to create an autoscale profile with min/max/default counts using `az monitor autoscale create`.
- How to add CPU-based scale-out and scale-in rules with `az monitor autoscale rule create`.
- Why cool-down periods and paired rules keep scaling stable.
- How to update the instance SKU and roll the change to existing instances.
