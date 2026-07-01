# Lab 24 — Azure Network Watcher and Connection Monitor

Maps to AZ-104 objective **Monitor and maintain Azure resources → Monitor networks with Azure Network Watcher** (use IP flow verify, next hop, NSG diagnostics, packet capture, and configure Connection Monitor).

You will deploy two VMs, then use Network Watcher's diagnostic tools to prove whether traffic is allowed, trace routing, capture packets, and continuously monitor reachability between endpoints — the network-troubleshooting skills the exam tests.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Azure Network Watcher (IP Flow Verify, Next Hop, NSG Diagnostics, Packet Capture, Connection Monitor), Virtual Network, NSG, Virtual Machines, a storage account for capture output.

---

## Step 1 — Create the resource group and network

**Portal:** Create **Resource group** `rg-az104-lab24`. Create a **Virtual network** `vnet-az104-lab24` with a `subnet-app` (10.10.1.0/24). Network Watcher is enabled per-region automatically.

```bash
LOCATION="southeastasia"
RG="rg-az104-lab24"

az group create --name "$RG" --location "$LOCATION"

az network vnet create \
  --resource-group "$RG" \
  --name "vnet-az104-lab24" \
  --address-prefix 10.10.0.0/16 \
  --subnet-name "subnet-app" \
  --subnet-prefix 10.10.1.0/24

# Network Watcher is auto-enabled per region; ensure it exists
az network watcher configure --resource-group NetworkWatcherRG --locations "$LOCATION" --enabled true 2>/dev/null || true
```

```powershell
$RG="rg-az104-lab24"; $LOCATION="southeastasia"
New-AzResourceGroup -Name $RG -Location $LOCATION
```

## Step 2 — Deploy two VMs with an NSG

**Portal:** Create two Ubuntu VMs (`vm-web`, `vm-app`) in `subnet-app`. Attach an NSG to `vm-web` allowing SSH (22) and HTTP (80). We'll test connectivity between them.

```bash
# vm-web with an NSG (SSH allowed by default rule set)
az vm create -g "$RG" -n "vm-web" --image Ubuntu2204 --size Standard_B1s \
  --vnet-name "vnet-az104-lab24" --subnet "subnet-app" \
  --admin-username azureuser --generate-ssh-keys --public-ip-sku Standard --nsg "nsg-web"

# open HTTP inbound on the vm-web NSG
az network nsg rule create -g "$RG" --nsg-name "nsg-web" -n "Allow-HTTP" \
  --priority 200 --access Allow --protocol Tcp --direction Inbound \
  --destination-port-ranges 80 --source-address-prefixes '*'

# vm-app (no public IP; internal only)
az vm create -g "$RG" -n "vm-app" --image Ubuntu2204 --size Standard_B1s \
  --vnet-name "vnet-az104-lab24" --subnet "subnet-app" \
  --admin-username azureuser --generate-ssh-keys --public-ip-address "" --nsg ""
```

## Step 3 — IP flow verify

**Portal:** **Network Watcher → IP flow verify**. Select `vm-web`, direction **Inbound**, protocol **TCP**, local port **80**, remote IP/port `0.0.0.0`/`50000`. The result says **Allow/Deny** and names the NSG rule responsible.

```bash
VMWEB_ID=$(az vm show -g "$RG" -n "vm-web" --query id -o tsv)

# is inbound TCP:80 from the internet allowed to vm-web? Returns the deciding NSG rule
az network watcher test-ip-flow \
  --vm "$VMWEB_ID" \
  --direction Inbound \
  --protocol TCP \
  --local "10.10.1.4:80" \
  --remote "13.68.0.1:50000"
```

```powershell
$nw = Get-AzNetworkWatcher -Location "southeastasia"
Test-AzNetworkWatcherIPFlow -NetworkWatcher $nw -TargetVirtualMachineId (Get-AzVM -ResourceGroupName $RG -Name "vm-web").Id `
  -Direction Inbound -Protocol TCP -LocalIPAddress 10.10.1.4 -LocalPort 80 -RemoteIPAddress 13.68.0.1 -RemotePort 50000
```

> IP flow verify answers "is this specific 5-tuple allowed?" and points at the exact effective NSG rule.

## Step 4 — Next hop

**Portal:** **Network Watcher → Next hop**. Source `vm-web`, source IP `10.10.1.4`, destination `8.8.8.8`. The result shows the next hop type (**Internet**, **VirtualAppliance**, **VnetLocal**, **None**) — great for diagnosing UDR/routing problems.

```bash
az network watcher show-next-hop \
  --resource-group "$RG" \
  --vm "vm-web" \
  --source-ip 10.10.1.4 \
  --dest-ip 8.8.8.8
```

```powershell
Get-AzNetworkWatcherNextHop -NetworkWatcher $nw `
  -TargetVirtualMachineId (Get-AzVM -ResourceGroupName $RG -Name "vm-web").Id `
  -SourceIPAddress 10.10.1.4 -DestinationIPAddress 8.8.8.8
```

## Step 5 — NSG diagnostics (security group view)

**Portal:** **Network Watcher → NSG diagnostics**. Pick `vm-web`, define the traffic (Inbound TCP:80 from Any) and run it to see which rule matches. Also use **Effective security rules** on the VM's NIC to view the merged NSG rule set.

```bash
NIC_ID=$(az vm show -g "$RG" -n "vm-web" --query "networkProfile.networkInterfaces[0].id" -o tsv)

# effective (merged subnet + NIC) NSG rules applied to the interface
az network nic list-effective-nsg --ids "$NIC_ID" -o jsonc

# NSG diagnostics: evaluate a specific flow against the security group
az network watcher nsg-flow-log show --location "$LOCATION" 2>/dev/null || true
```

> "Effective security rules" is the merged view of subnet-level + NIC-level NSGs — the exam loves questions where a subnet NSG overrides what you expected.

## Step 6 — Enable NSG flow logs (traffic analytics)

**Portal:** **Network Watcher → NSG flow logs → + Create**. Target `nsg-web`, choose a storage account, optionally send to a Log Analytics workspace to enable **Traffic Analytics**. Flow logs record allowed/denied flows over time.

```bash
STORAGE="stlab24$RANDOM"
az storage account create -g "$RG" -n "$STORAGE" --sku Standard_LRS --location "$LOCATION"
STORAGE_ID=$(az storage account show -g "$RG" -n "$STORAGE" --query id -o tsv)
NSG_ID=$(az network nsg show -g "$RG" -n "nsg-web" --query id -o tsv)

az network watcher flow-log create \
  --location "$LOCATION" \
  --name "flowlog-nsg-web" \
  --nsg "$NSG_ID" \
  --storage-account "$STORAGE_ID" \
  --retention 7 \
  --enabled true
```

## Step 7 — Packet capture

**Portal:** **Network Watcher → Packet capture → + Add**. Target `vm-web`, set a name, storage account and a size/time limit, then **Start**. Download the `.cap` file when done and open in Wireshark. (Packet capture needs the **AzureNetworkWatcher** VM extension, which the portal installs automatically.)

```bash
# install the network watcher agent extension (required for packet capture)
az vm extension set -g "$RG" --vm-name "vm-web" \
  --name NetworkWatcherAgentLinux --publisher Microsoft.Azure.NetworkWatcher --version 1.4

# start a 60-second capture to the storage account
az network watcher packet-capture create \
  --resource-group "$RG" \
  --vm "vm-web" \
  --name "pcap-vm-web" \
  --storage-account "$STORAGE" \
  --time-limit 60

# check status, then stop and download from the storage account
az network watcher packet-capture show-status --name "pcap-vm-web" --location "$LOCATION"
az network watcher packet-capture stop --name "pcap-vm-web" --location "$LOCATION"
```

## Step 8 — Connection troubleshoot (point-in-time)

**Portal:** **Network Watcher → Connection troubleshoot**. Source `vm-web`, destination `vm-app` on port 22 (or destination URI). It reports reachability, latency, and any blocking hop.

```bash
VMAPP_IP=$(az vm show -g "$RG" -n "vm-app" -d --query privateIps -o tsv)

# one-off connectivity test from vm-web to vm-app over SSH
az network watcher test-connectivity \
  --resource-group "$RG" \
  --source-resource "vm-web" \
  --dest-address "$VMAPP_IP" \
  --dest-port 22
```

```powershell
Test-AzNetworkWatcherConnectivity -NetworkWatcher $nw `
  -SourceId (Get-AzVM -ResourceGroupName $RG -Name "vm-web").Id `
  -DestinationAddress "10.10.1.5" -DestinationPort 22
```

## Step 9 — Configure Connection Monitor (continuous)

**Portal:** **Network Watcher → Connection monitor → + Create**. Name `cm-web-to-app`, add a **test group** with source `vm-web` and destination `vm-app` (or an external URL), TCP:22, then **Create**. Connection Monitor continuously probes reachability/latency and can alert on degradation, feeding results into Log Analytics.

```bash
# create a Connection Monitor (v2) with a source/destination test
az network watcher connection-monitor create \
  --name "cm-web-to-app" \
  --location "$LOCATION" \
  --endpoint-source-name "src-vm-web" \
  --endpoint-source-resource-id "$VMWEB_ID" \
  --endpoint-dest-name "dst-vm-app" \
  --endpoint-dest-address "$VMAPP_IP" \
  --test-config-name "tcp-22" \
  --protocol Tcp \
  --tcp-port 22 \
  --frequency 30

# show collected results / status
az network watcher connection-monitor show --name "cm-web-to-app" --location "$LOCATION" -o jsonc
```

> Connection troubleshoot = one-shot diagnostic. **Connection Monitor** = ongoing synthetic monitoring with history and alerting — know when to use each.

## Clean up

```bash
# VMs, NICs, NSG, VNet, storage, flow log config and packet captures live in the RG
az group delete --name rg-az104-lab24 --yes --no-wait
```

> Connection Monitors and NSG flow-log configurations attached to resources in this RG are removed with it. Note that Network Watcher itself is a regional service in the auto-created `NetworkWatcherRG` — leave that in place; it is not billed on its own.

## References

- Azure Network Watcher overview — https://learn.microsoft.com/azure/network-watcher/network-watcher-overview
- Diagnose network traffic filtering (IP flow verify) — https://learn.microsoft.com/azure/network-watcher/diagnose-vm-network-traffic-filtering-problem
- Packet capture overview — https://learn.microsoft.com/azure/network-watcher/packet-capture-overview
- Connection Monitor overview — https://learn.microsoft.com/azure/network-watcher/connection-monitor-overview

## What you learned

- Use **IP flow verify** to prove whether a 5-tuple is allowed and identify the deciding NSG rule.
- Trace routing decisions with **Next hop** and read **effective security rules** for NSG diagnostics.
- Enable **NSG flow logs** and capture live traffic with **packet capture** for offline analysis.
- Run a point-in-time **Connection troubleshoot** and set up continuous **Connection Monitor**.
- Distinguish one-off diagnostics from continuous synthetic monitoring, and where results are stored.
- Perform every task via Portal, Azure CLI, and Azure PowerShell.
