# Lab 18 — Virtual Networks, Subnets & VNet Peering

Maps to AZ-104 objective **Implement and manage virtual networking → Configure virtual networks** (create and configure virtual networks and subnets; create and configure virtual network peering).

You will create two virtual networks with subnets in different address spaces, then peer them so resources in each VNet can communicate over the Azure backbone without a VPN or public IPs.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Azure Virtual Network (VNet), Subnets, VNet Peering, Network Watcher.

---

## Step 1 — Create the resource group

**Portal:** Home → **Resource groups** → **+ Create** → Subscription, Name `rg-az104-lab18`, Region **East US** → **Review + create** → **Create**.

```bash
az group create --name rg-az104-lab18 --location eastus
```

```powershell
New-AzResourceGroup -Name rg-az104-lab18 -Location eastus
```

## Step 2 — Create the first virtual network with a subnet

**Portal:** **+ Create a resource** → **Networking** → **Virtual network** → RG `rg-az104-lab18`, Name `vnet-hub`, Region East US → **IP addresses** tab: address space `10.10.0.0/16`, add subnet `snet-hub-app` `10.10.1.0/24` → **Review + create**.

```bash
az network vnet create \
  --resource-group rg-az104-lab18 \
  --name vnet-hub \
  --address-prefixes 10.10.0.0/16 \
  --subnet-name snet-hub-app \
  --subnet-prefixes 10.10.1.0/24 \
  --location eastus
```

```powershell
$subHub = New-AzVirtualNetworkSubnetConfig -Name snet-hub-app -AddressPrefix 10.10.1.0/24
New-AzVirtualNetwork -ResourceGroupName rg-az104-lab18 -Name vnet-hub `
  -Location eastus -AddressPrefix 10.10.0.0/16 -Subnet $subHub
```

## Step 3 — Create the second virtual network with a subnet

**Portal:** Repeat Step 2 for `vnet-spoke` with address space `10.20.0.0/16` and subnet `snet-spoke-app` `10.20.1.0/24`. Address spaces must **not overlap** or peering will fail.

```bash
az network vnet create \
  --resource-group rg-az104-lab18 \
  --name vnet-spoke \
  --address-prefixes 10.20.0.0/16 \
  --subnet-name snet-spoke-app \
  --subnet-prefixes 10.20.1.0/24 \
  --location eastus
```

## Step 4 — Add a second subnet to the hub VNet

**Portal:** `vnet-hub` → **Subnets** → **+ Subnet** → Name `snet-hub-mgmt`, range `10.10.2.0/24` → **Save**.

```bash
az network vnet subnet create \
  --resource-group rg-az104-lab18 \
  --vnet-name vnet-hub \
  --name snet-hub-mgmt \
  --address-prefixes 10.10.2.0/24
```

```powershell
$vnet = Get-AzVirtualNetwork -ResourceGroupName rg-az104-lab18 -Name vnet-hub
Add-AzVirtualNetworkSubnetConfig -Name snet-hub-mgmt -AddressPrefix 10.10.2.0/24 -VirtualNetwork $vnet
$vnet | Set-AzVirtualNetwork
```

## Step 5 — Create the peering from hub to spoke

**Portal:** `vnet-hub` → **Peerings** → **+ Add**. This link name `hub-to-spoke`; remote link name `spoke-to-hub`; peer VNet `vnet-spoke`. Leave **Allow traffic to remote virtual network** enabled. The Portal creates both directions in one step.

```bash
# Direction 1: hub -> spoke
az network vnet peering create \
  --resource-group rg-az104-lab18 \
  --name hub-to-spoke \
  --vnet-name vnet-hub \
  --remote-vnet vnet-spoke \
  --allow-vnet-access
```

## Step 6 — Create the reverse peering from spoke to hub

**Portal:** (already created if you used the Portal in Step 5). Via CLI, peering must be created **on both sides** — otherwise the state stays `Initiated` instead of `Connected`.

```bash
# Direction 2: spoke -> hub
az network vnet peering create \
  --resource-group rg-az104-lab18 \
  --name spoke-to-hub \
  --vnet-name vnet-spoke \
  --remote-vnet vnet-hub \
  --allow-vnet-access
```

```powershell
$hub   = Get-AzVirtualNetwork -ResourceGroupName rg-az104-lab18 -Name vnet-hub
$spoke = Get-AzVirtualNetwork -ResourceGroupName rg-az104-lab18 -Name vnet-spoke
Add-AzVirtualNetworkPeering -Name hub-to-spoke -VirtualNetwork $hub  -RemoteVirtualNetworkId $spoke.Id -AllowForwardedTraffic
Add-AzVirtualNetworkPeering -Name spoke-to-hub -VirtualNetwork $spoke -RemoteVirtualNetworkId $hub.Id  -AllowForwardedTraffic
```

## Step 7 — Verify the peering state is Connected

**Portal:** `vnet-hub` → **Peerings** → confirm **Peering status = Connected** on both entries.

```bash
az network vnet peering show \
  --resource-group rg-az104-lab18 \
  --vnet-name vnet-hub \
  --name hub-to-spoke \
  --query "{name:name, state:peeringState, allowAccess:allowVirtualNetworkAccess}" -o table
# Both peerings must show peeringState = Connected
```

## Step 8 — (Optional) Deploy test VMs and verify connectivity

**Portal:** Deploy a small VM into `snet-hub-app` and another into `snet-spoke-app`, then use **Network Watcher → Connection troubleshoot** to ping across the peering.

```bash
# Create two tiny Linux VMs (no public IP needed for the peering test)
az vm create -g rg-az104-lab18 -n vm-hub   --image Ubuntu2204 --size Standard_B1s \
  --vnet-name vnet-hub   --subnet snet-hub-app   --public-ip-address "" --admin-username azureuser --generate-ssh-keys
az vm create -g rg-az104-lab18 -n vm-spoke --image Ubuntu2204 --size Standard_B1s \
  --vnet-name vnet-spoke --subnet snet-spoke-app --public-ip-address "" --admin-username azureuser --generate-ssh-keys

# Test connectivity across the peering with Network Watcher
az network watcher test-connectivity \
  --resource-group rg-az104-lab18 \
  --source-resource vm-hub \
  --dest-resource vm-spoke \
  --dest-port 22
# ConnectionStatus should be "Reachable"
```

## Clean up

Delete everything so you stop incurring VM and networking charges.

```bash
az group delete --name rg-az104-lab18 --yes --no-wait
```

> **Cost warning:** VMs, public IPs, load balancers, gateways and Bastion bill hourly. VNets and peering configuration have no standby charge, but peered data transfer is billed per GB. Delete promptly when finished.

## References
- Virtual network peering overview — https://learn.microsoft.com/azure/virtual-network/virtual-network-peering-overview
- Create, change, or delete a peering — https://learn.microsoft.com/azure/virtual-network/virtual-network-manage-peering
- Quickstart: Create a virtual network — https://learn.microsoft.com/azure/virtual-network/quick-create-cli

## What you learned
- How to create VNets with non-overlapping address spaces and multiple subnets.
- Why address spaces must not overlap for peering to succeed.
- That VNet peering is a two-sided link — both directions must be created and reach `Connected`.
- The role of `--allow-vnet-access` and forwarded-traffic flags on a peering.
- How to validate cross-VNet reachability with Network Watcher `test-connectivity`.
- That peered traffic stays on the Microsoft backbone (no gateway/VPN required for same-region peering).
