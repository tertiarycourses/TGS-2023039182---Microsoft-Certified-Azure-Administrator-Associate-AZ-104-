# Lab 19 — Public IPs, User-Defined Routes & Connectivity Troubleshooting

Maps to AZ-104 objective **Implement and manage virtual networking → Configure IP addressing and routing** (configure public IP addresses; configure user-defined routes / route tables; troubleshoot network connectivity).

You will allocate static and dynamic public IPs, build a route table that forces subnet traffic through a virtual appliance (a UDR), associate it to a subnet, then troubleshoot the resulting effective routes.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Public IP address, Route Table (User-Defined Routes), Virtual Network, Network Watcher, Network Interface (NIC).

---

## Step 1 — Create the resource group and a VNet

**Portal:** **Resource groups** → **+ Create** → Name `rg-az104-lab19`, Region East US. Then create VNet `vnet-lab19` (`10.30.0.0/16`) with subnet `snet-workload` (`10.30.1.0/24`).

```bash
az group create --name rg-az104-lab19 --location eastus

az network vnet create \
  --resource-group rg-az104-lab19 \
  --name vnet-lab19 \
  --address-prefixes 10.30.0.0/16 \
  --subnet-name snet-workload \
  --subnet-prefixes 10.30.1.0/24
```

## Step 2 — Create a static Standard SKU public IP

**Portal:** **+ Create a resource** → **Public IP address** → Name `pip-web-static`, SKU **Standard**, Assignment **Static**, RG `rg-az104-lab19`. Standard SKU is always static and zone-redundant.

```bash
az network public-ip create \
  --resource-group rg-az104-lab19 \
  --name pip-web-static \
  --sku Standard \
  --allocation-method Static \
  --version IPv4
```

```powershell
New-AzPublicIpAddress -ResourceGroupName rg-az104-lab19 -Name pip-web-static `
  -Location eastus -Sku Standard -AllocationMethod Static -IpAddressVersion IPv4
```

## Step 3 — Inspect the allocated address

**Portal:** `pip-web-static` → **Overview** → note the **IP address** and **DNS name**.

```bash
az network public-ip show \
  --resource-group rg-az104-lab19 \
  --name pip-web-static \
  --query "{ip:ipAddress, sku:sku.name, alloc:publicIPAllocationMethod}" -o table
```

## Step 4 — Create a route table

**Portal:** **+ Create a resource** → search **Route table** → Name `rt-workload`, RG `rg-az104-lab19`, Region East US, **Propagate gateway routes = Yes** → Create.

```bash
az network route-table create \
  --resource-group rg-az104-lab19 \
  --name rt-workload \
  --location eastus
```

```powershell
New-AzRouteTable -ResourceGroupName rg-az104-lab19 -Name rt-workload -Location eastus
```

## Step 5 — Add a user-defined route to a virtual appliance

**Portal:** `rt-workload` → **Routes** → **+ Add**. Name `route-to-nva`, Address prefix `0.0.0.0/0`, Next hop type **Virtual appliance**, Next hop address `10.30.1.4` (your firewall/NVA NIC). This forces all internet-bound traffic through the appliance.

```bash
az network route-table route create \
  --resource-group rg-az104-lab19 \
  --route-table-name rt-workload \
  --name route-to-nva \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.30.1.4
```

```powershell
$rt = Get-AzRouteTable -ResourceGroupName rg-az104-lab19 -Name rt-workload
Add-AzRouteConfig -RouteTable $rt -Name route-to-nva `
  -AddressPrefix 0.0.0.0/0 -NextHopType VirtualAppliance -NextHopIpAddress 10.30.1.4 | Set-AzRouteTable
```

## Step 6 — Associate the route table to the subnet

**Portal:** `rt-workload` → **Subnets** → **+ Associate** → VNet `vnet-lab19`, Subnet `snet-workload` → OK. A subnet can have only **one** route table.

```bash
az network vnet subnet update \
  --resource-group rg-az104-lab19 \
  --vnet-name vnet-lab19 \
  --name snet-workload \
  --route-table rt-workload
```

```powershell
$vnet = Get-AzVirtualNetwork -ResourceGroupName rg-az104-lab19 -Name vnet-lab19
$rt   = Get-AzRouteTable -ResourceGroupName rg-az104-lab19 -Name rt-workload
Set-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name snet-workload `
  -AddressPrefix 10.30.1.0/24 -RouteTable $rt | Set-AzVirtualNetwork
```

## Step 7 — Deploy a test VM and attach the static public IP

**Portal:** Create VM `vm-web` in `snet-workload`, and on the **Networking** tab select the existing public IP `pip-web-static`.

```bash
az vm create \
  --resource-group rg-az104-lab19 \
  --name vm-web \
  --image Ubuntu2204 --size Standard_B1s \
  --vnet-name vnet-lab19 --subnet snet-workload \
  --public-ip-address pip-web-static \
  --admin-username azureuser --generate-ssh-keys
```

## Step 8 — View the effective routes on the NIC

**Portal:** `vm-web` → **Networking** → select the NIC → **Effective routes** (under Help/Support). Confirm your `0.0.0.0/0 → VirtualAppliance` route is present and overriding the default `Internet` route.

```bash
# Get the NIC name, then show its effective route table
NIC=$(az vm show -g rg-az104-lab19 -n vm-web --query "networkProfile.networkInterfaces[0].id" -o tsv | xargs -I{} basename {})
az network nic show-effective-route-table \
  --resource-group rg-az104-lab19 \
  --name "$NIC" -o table
# Look for source "User" with next-hop "VirtualAppliance" for 0.0.0.0/0
```

```powershell
$nic = (Get-AzVM -ResourceGroupName rg-az104-lab19 -Name vm-web).NetworkProfile.NetworkInterfaces[0].Id
Get-AzEffectiveRouteTable -ResourceGroupName rg-az104-lab19 -NetworkInterfaceName (Split-Path $nic -Leaf) | Format-Table
```

## Step 9 — Troubleshoot connectivity with Network Watcher

**Portal:** **Network Watcher** → **Connection troubleshoot** → Source `vm-web`, Destination URI `https://www.microsoft.com` → **Check**. Because traffic is forced to a (non-existent) NVA, expect it to be **Unreachable** — proving the UDR is in effect.

```bash
# IP flow verify: does the UDR/NSG allow this outbound flow?
az network watcher test-connectivity \
  --resource-group rg-az104-lab19 \
  --source-resource vm-web \
  --dest-address 13.107.42.14 --dest-port 443
# ConnectionStatus "Unreachable" here confirms the 0.0.0.0/0 UDR is diverting traffic.
```

## Clean up

```bash
az group delete --name rg-az104-lab19 --yes --no-wait
```

> **Cost warning:** Standard public IPs, VMs, gateways, load balancers and Bastion bill hourly. Standard static public IPs are charged even when not attached to a running resource — delete promptly.

## References
- Public IP addresses in Azure — https://learn.microsoft.com/azure/virtual-network/ip-services/public-ip-addresses
- Virtual network traffic routing (UDRs) — https://learn.microsoft.com/azure/virtual-network/virtual-networks-udr-overview
- Diagnose a VM routing problem — https://learn.microsoft.com/azure/network-watcher/diagnose-vm-network-routing-problem
- Connection troubleshoot overview — https://learn.microsoft.com/azure/network-watcher/network-watcher-connectivity-overview

## What you learned
- The difference between Basic/Standard SKU public IPs and dynamic vs static allocation.
- How to build a route table and add a user-defined route with a virtual-appliance next hop.
- That a subnet can have exactly one associated route table, and UDRs override Azure system routes.
- How to read a NIC's effective routes to confirm which route wins.
- How to use Network Watcher connection troubleshoot / test-connectivity to prove routing behavior.
- Why a `0.0.0.0/0` UDR can silently break internet egress if the next-hop appliance is missing.
