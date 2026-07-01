# Lab 20 — Network Security Groups, Application Security Groups & Effective Rules

Maps to AZ-104 objective **Implement and manage virtual networking → Secure network traffic** (create and configure NSGs and ASGs; associate NSGs to subnets and NICs; evaluate effective security rules).

You will create an NSG with custom rules, group VMs by role using application security groups (ASGs), reference those ASGs in NSG rules, then read the effective security rules Azure computes for a NIC.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Network Security Group (NSG), Application Security Group (ASG), Virtual Network, Network Interface, Network Watcher.

---

## Step 1 — Create the resource group, VNet and subnet

**Portal:** **Resource groups** → **+ Create** → `rg-az104-lab20`, East US. Then VNet `vnet-lab20` (`10.40.0.0/16`) with subnet `snet-app` (`10.40.1.0/24`).

```bash
az group create --name rg-az104-lab20 --location eastus

az network vnet create \
  --resource-group rg-az104-lab20 \
  --name vnet-lab20 \
  --address-prefixes 10.40.0.0/16 \
  --subnet-name snet-app \
  --subnet-prefixes 10.40.1.0/24
```

## Step 2 — Create application security groups

**Portal:** **+ Create a resource** → search **Application security group** → create `asg-web` and `asg-db` in `rg-az104-lab20`, East US.

```bash
az network asg create --resource-group rg-az104-lab20 --name asg-web --location eastus
az network asg create --resource-group rg-az104-lab20 --name asg-db  --location eastus
```

```powershell
New-AzApplicationSecurityGroup -ResourceGroupName rg-az104-lab20 -Name asg-web -Location eastus
New-AzApplicationSecurityGroup -ResourceGroupName rg-az104-lab20 -Name asg-db  -Location eastus
```

## Step 3 — Create the network security group

**Portal:** **+ Create a resource** → **Network security group** → Name `nsg-app`, RG `rg-az104-lab20`, East US.

```bash
az network nsg create --resource-group rg-az104-lab20 --name nsg-app --location eastus
```

```powershell
New-AzNetworkSecurityGroup -ResourceGroupName rg-az104-lab20 -Name nsg-app -Location eastus
```

## Step 4 — Allow HTTPS from the internet to the web ASG

**Portal:** `nsg-app` → **Inbound security rules** → **+ Add** → Source **Any**, Destination **Application security group** → `asg-web`, Destination port **443**, Protocol TCP, Action **Allow**, Priority **100**, Name `Allow-HTTPS-Web`.

```bash
az network nsg rule create \
  --resource-group rg-az104-lab20 \
  --nsg-name nsg-app \
  --name Allow-HTTPS-Web \
  --priority 100 \
  --direction Inbound --access Allow --protocol Tcp \
  --source-address-prefixes Internet --source-port-ranges '*' \
  --destination-asgs asg-web --destination-port-ranges 443
```

```powershell
$web = Get-AzApplicationSecurityGroup -ResourceGroupName rg-az104-lab20 -Name asg-web
$nsg = Get-AzNetworkSecurityGroup -ResourceGroupName rg-az104-lab20 -Name nsg-app
$nsg | Add-AzNetworkSecurityRuleConfig -Name Allow-HTTPS-Web -Priority 100 `
  -Direction Inbound -Access Allow -Protocol Tcp `
  -SourceAddressPrefix Internet -SourcePortRange * `
  -DestinationApplicationSecurityGroup $web -DestinationPortRange 443 | Set-AzNetworkSecurityGroup
```

## Step 5 — Allow SQL only from the web ASG to the db ASG

**Portal:** Add inbound rule `Allow-SQL-Web-to-DB` → Source **Application security group** `asg-web`, Destination **Application security group** `asg-db`, port **1433**, Allow, Priority **110**.

```bash
az network nsg rule create \
  --resource-group rg-az104-lab20 \
  --nsg-name nsg-app \
  --name Allow-SQL-Web-to-DB \
  --priority 110 \
  --direction Inbound --access Allow --protocol Tcp \
  --source-asgs asg-web --source-port-ranges '*' \
  --destination-asgs asg-db --destination-port-ranges 1433
# ASG-to-ASG rules let you write intent-based policy independent of IP addresses.
```

## Step 6 — Explicitly deny everything else inbound

**Portal:** Add rule `Deny-All-Inbound` → Source Any, Destination Any, port Any, Action **Deny**, Priority **4000** (below your allow rules; above the default 65500).

```bash
az network nsg rule create \
  --resource-group rg-az104-lab20 \
  --nsg-name nsg-app \
  --name Deny-All-Inbound \
  --priority 4000 \
  --direction Inbound --access Deny --protocol '*' \
  --source-address-prefixes '*' --source-port-ranges '*' \
  --destination-address-prefixes '*' --destination-port-ranges '*'
```

## Step 7 — Associate the NSG to the subnet

**Portal:** `nsg-app` → **Subnets** → **+ Associate** → `vnet-lab20` / `snet-app`.

```bash
az network vnet subnet update \
  --resource-group rg-az104-lab20 \
  --vnet-name vnet-lab20 \
  --name snet-app \
  --network-security-group nsg-app
```

## Step 8 — Deploy VMs and place their NICs in the ASGs

**Portal:** Create `vm-web` and `vm-db` in `snet-app`. On each VM's NIC → **Application security groups** → add `asg-web` / `asg-db` respectively.

```bash
az vm create -g rg-az104-lab20 -n vm-web --image Ubuntu2204 --size Standard_B1s \
  --vnet-name vnet-lab20 --subnet snet-app --nsg "" --admin-username azureuser --generate-ssh-keys
az vm create -g rg-az104-lab20 -n vm-db  --image Ubuntu2204 --size Standard_B1s \
  --vnet-name vnet-lab20 --subnet snet-app --nsg "" --public-ip-address "" --admin-username azureuser --generate-ssh-keys

# Put each VM's NIC into its ASG
WEB_NIC=$(az vm show -g rg-az104-lab20 -n vm-web --query "networkProfile.networkInterfaces[0].id" -o tsv | xargs -I{} basename {})
DB_NIC=$(az vm show -g rg-az104-lab20 -n vm-db  --query "networkProfile.networkInterfaces[0].id" -o tsv | xargs -I{} basename {})
az network nic ip-config update -g rg-az104-lab20 --nic-name "$WEB_NIC" -n ipconfig1 --application-security-groups asg-web
az network nic ip-config update -g rg-az104-lab20 --nic-name "$DB_NIC"  -n ipconfig1 --application-security-groups asg-db
```

## Step 9 — Evaluate the effective security rules

**Portal:** `vm-web` → **Networking** → NIC → **Effective security rules**. Confirm your allow/deny rules plus the platform defaults are merged, and that the ASG names resolve to the NIC's private IPs.

```bash
az network nic list-effective-nsg \
  --resource-group rg-az104-lab20 \
  --name "$WEB_NIC" -o table
# The effective view combines subnet NSG + NIC NSG + default rules, ordered by priority.
```

```powershell
Get-AzEffectiveNetworkSecurityGroup -ResourceGroupName rg-az104-lab20 -NetworkInterfaceName $WEB_NIC
```

## Step 10 — Validate a flow with IP flow verify

**Portal:** **Network Watcher** → **IP flow verify** → VM `vm-web`, Direction Inbound, Protocol TCP, Local port 443, Remote IP `1.2.3.4:12345` → **Check** → expect **Allow** by `Allow-HTTPS-Web`. Repeat for port 22 → expect **Deny**.

```bash
az network watcher test-ip-flow \
  --resource-group rg-az104-lab20 \
  --vm vm-web \
  --direction Inbound --protocol TCP \
  --local 10.40.1.4:443 --remote 1.2.3.4:33000
# Access "Allow", RuleName "Allow-HTTPS-Web". Try local port 22 to see a Deny.
```

## Clean up

```bash
az group delete --name rg-az104-lab20 --yes --no-wait
```

> **Cost warning:** VMs, public IPs, load balancers, gateways and Bastion bill hourly. NSGs and ASGs themselves are free, but the VMs behind them are not — delete promptly.

## References
- Network security groups overview — https://learn.microsoft.com/azure/virtual-network/network-security-groups-overview
- Application security groups — https://learn.microsoft.com/azure/virtual-network/application-security-groups
- How Azure evaluates effective security rules — https://learn.microsoft.com/azure/virtual-network/network-security-group-how-it-works
- Diagnose a VM traffic filter problem — https://learn.microsoft.com/azure/network-watcher/diagnose-vm-network-traffic-filtering-problem

## What you learned
- How to create NSGs and add prioritized inbound/outbound allow and deny rules.
- How ASGs let you write rules by workload role instead of hard-coded IP ranges.
- The difference between associating an NSG to a subnet versus a NIC, and how both combine.
- That lower priority numbers win, and a default deny sits at 65500.
- How to read effective security rules to see the merged, evaluated rule set for a NIC.
- How to use Network Watcher IP flow verify to prove whether a specific flow is allowed or denied.
