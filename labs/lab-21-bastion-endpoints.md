# Lab 21 — Azure Bastion, Service Endpoints & Private Endpoints

Maps to AZ-104 objective **Implement and manage virtual networking → Provide secure access to PaaS and VMs** (implement Azure Bastion; configure service endpoints for Azure PaaS; configure private endpoints for Azure PaaS).

You will deploy Azure Bastion for browser-based RDP/SSH without public IPs on VMs, lock a storage account down to a subnet using a service endpoint, then bring that storage into the VNet with a private endpoint and private DNS.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Azure Bastion, Virtual Network, Public IP, Service Endpoints, Private Endpoint, Private DNS Zone, Azure Storage.

---

## Step 1 — Create the resource group and VNet

**Portal:** `rg-az104-lab21`, East US. VNet `vnet-lab21` (`10.50.0.0/16`) with subnet `snet-workload` (`10.50.1.0/24`).

```bash
az group create --name rg-az104-lab21 --location eastus

az network vnet create \
  --resource-group rg-az104-lab21 \
  --name vnet-lab21 \
  --address-prefixes 10.50.0.0/16 \
  --subnet-name snet-workload \
  --subnet-prefixes 10.50.1.0/24
```

## Step 2 — Add the AzureBastionSubnet

**Portal:** `vnet-lab21` → **Subnets** → **+ Subnet**. The subnet **must** be named exactly `AzureBastionSubnet` and be **/26 or larger**.

```bash
az network vnet subnet create \
  --resource-group rg-az104-lab21 \
  --vnet-name vnet-lab21 \
  --name AzureBastionSubnet \
  --address-prefixes 10.50.2.0/26
# Name is case-sensitive and mandatory; /26 minimum.
```

## Step 3 — Create a Standard public IP for Bastion

**Portal:** Create Public IP `pip-bastion`, SKU **Standard**, Static. Bastion requires a Standard SKU public IP.

```bash
az network public-ip create \
  --resource-group rg-az104-lab21 \
  --name pip-bastion \
  --sku Standard --allocation-method Static
```

## Step 4 — Deploy Azure Bastion

**Portal:** **+ Create a resource** → **Bastion** → Name `bastion-lab21`, VNet `vnet-lab21`, Subnet `AzureBastionSubnet`, Public IP `pip-bastion` → Create (deployment takes several minutes).

```bash
az network bastion create \
  --resource-group rg-az104-lab21 \
  --name bastion-lab21 \
  --vnet-name vnet-lab21 \
  --public-ip-address pip-bastion \
  --location eastus \
  --sku Standard
```

```powershell
$pip  = Get-AzPublicIpAddress -ResourceGroupName rg-az104-lab21 -Name pip-bastion
$vnet = Get-AzVirtualNetwork -ResourceGroupName rg-az104-lab21 -Name vnet-lab21
New-AzBastion -ResourceGroupName rg-az104-lab21 -Name bastion-lab21 `
  -PublicIpAddress $pip -VirtualNetwork $vnet -Sku Standard
```

## Step 5 — Deploy a VM with no public IP and connect via Bastion

**Portal:** Create `vm-jump` in `snet-workload` with **no public IP**. Then `vm-jump` → **Connect** → **Bastion** → enter credentials → connect in-browser.

```bash
az vm create \
  --resource-group rg-az104-lab21 \
  --name vm-jump --image Ubuntu2204 --size Standard_B1s \
  --vnet-name vnet-lab21 --subnet snet-workload \
  --public-ip-address "" \
  --admin-username azureuser --generate-ssh-keys
# Connect from the Portal: vm-jump > Connect > Bastion (no public IP on the VM).
```

## Step 6 — Create a storage account for the endpoint tests

**Portal:** **+ Create a resource** → **Storage account** → Name `stlab21<unique>`, RG `rg-az104-lab21`, Standard LRS.

```bash
STG=stlab21$RANDOM
az storage account create \
  --resource-group rg-az104-lab21 \
  --name $STG --sku Standard_LRS --kind StorageV2 --location eastus
echo $STG
```

## Step 7 — Enable a service endpoint on the subnet

**Portal:** `vnet-lab21` → `snet-workload` → **Service endpoints** → add **Microsoft.Storage** → Save. This tags subnet traffic so storage can trust it by VNet rule.

```bash
az network vnet subnet update \
  --resource-group rg-az104-lab21 \
  --vnet-name vnet-lab21 \
  --name snet-workload \
  --service-endpoints Microsoft.Storage
```

```powershell
$vnet = Get-AzVirtualNetwork -ResourceGroupName rg-az104-lab21 -Name vnet-lab21
Set-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name snet-workload `
  -AddressPrefix 10.50.1.0/24 -ServiceEndpoint Microsoft.Storage | Set-AzVirtualNetwork
```

## Step 8 — Restrict the storage account to that subnet

**Portal:** Storage account → **Networking** → **Enabled from selected virtual networks** → add `vnet-lab21` / `snet-workload` → Save. Traffic still uses the storage public endpoint but is now VNet-scoped.

```bash
az storage account network-rule add \
  --resource-group rg-az104-lab21 \
  --account-name $STG \
  --vnet-name vnet-lab21 --subnet snet-workload

az storage account update \
  --resource-group rg-az104-lab21 --name $STG \
  --default-action Deny
# Now only the service-endpoint-enabled subnet (plus allowed IPs) can reach the account.
```

## Step 9 — Create a private DNS zone for blob storage

**Portal:** **+ Create a resource** → **Private DNS zone** → Name `privatelink.blob.core.windows.net` → then **Virtual network links** → **+ Add** → link `vnet-lab21`, enable auto-registration off.

```bash
az network private-dns zone create \
  --resource-group rg-az104-lab21 \
  --name privatelink.blob.core.windows.net

az network private-dns link vnet create \
  --resource-group rg-az104-lab21 \
  --zone-name privatelink.blob.core.windows.net \
  --name link-vnet-lab21 \
  --virtual-network vnet-lab21 \
  --registration-enabled false
```

## Step 10 — Create a private endpoint and integrate the DNS zone

**Portal:** Storage account → **Networking** → **Private endpoint connections** → **+ Private endpoint** → Name `pe-storage-blob`, Subnet `snet-workload`, target sub-resource **blob**, integrate with private DNS zone → Create.

```bash
STG_ID=$(az storage account show -g rg-az104-lab21 -n $STG --query id -o tsv)

az network private-endpoint create \
  --resource-group rg-az104-lab21 \
  --name pe-storage-blob \
  --vnet-name vnet-lab21 --subnet snet-workload \
  --private-connection-resource-id "$STG_ID" \
  --group-id blob \
  --connection-name pe-storage-conn

# Wire the private endpoint's IP into the private DNS zone
az network private-endpoint dns-zone-group create \
  --resource-group rg-az104-lab21 \
  --endpoint-name pe-storage-blob \
  --name pe-dns-group \
  --private-dns-zone privatelink.blob.core.windows.net \
  --zone-name blob
```

## Step 11 — Verify private name resolution from the VM

**Portal:** Connect to `vm-jump` via **Bastion** and run `nslookup <storage>.blob.core.windows.net` — it should resolve to a **10.50.x.x** private address, not a public IP.

```bash
# Run this inside vm-jump (via Bastion session):
nslookup ${STG}.blob.core.windows.net
# Expect the CNAME to privatelink.blob.core.windows.net -> a 10.50.1.x private IP.
```

## Clean up

```bash
az group delete --name rg-az104-lab21 --yes --no-wait
```

> **Cost warning:** Azure **Bastion**, load balancers, public IPs and gateways bill **hourly** — Bastion in particular is one of the pricier hourly items. Private endpoints also carry an hourly + per-GB charge. Delete this resource group promptly after the lab.

## References
- What is Azure Bastion? — https://learn.microsoft.com/azure/bastion/bastion-overview
- Virtual network service endpoints — https://learn.microsoft.com/azure/virtual-network/virtual-network-service-endpoints-overview
- What is a private endpoint? — https://learn.microsoft.com/azure/private-link/private-endpoint-overview
- Private endpoint DNS configuration — https://learn.microsoft.com/azure/private-link/private-endpoint-dns

## What you learned
- How to deploy Azure Bastion (requires an `AzureBastionSubnet` /26+ and a Standard public IP) for agentless RDP/SSH.
- How to connect to a VM that has no public IP entirely through the browser.
- The difference between a service endpoint (VNet-scoped access to the PaaS public endpoint) and a private endpoint (a private IP in your VNet).
- How to lock a storage account to a specific subnet with a service endpoint and default-deny.
- How private endpoints require a matching private DNS zone (`privatelink.*`) to resolve correctly.
- How to verify private name resolution from inside the VNet.
