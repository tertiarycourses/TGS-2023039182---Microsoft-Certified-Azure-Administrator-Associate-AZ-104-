# Lab 22 — Azure DNS & Load Balancing

Maps to AZ-104 objective **Implement and manage virtual networking → Configure name resolution and load balancing** (configure Azure DNS; configure an internal or public load balancer; troubleshoot load balancing).

You will host a public DNS zone with A and CNAME records, then build a public Standard load balancer with a backend pool, health probe and load-balancing rule across two VMs, and troubleshoot when traffic does not distribute.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Azure DNS (public zone), Azure Load Balancer (Standard), Public IP, Virtual Network, Backend Pool, Health Probe, Network Watcher.

---

## Step 1 — Create the resource group and VNet

**Portal:** `rg-az104-lab22`, East US. VNet `vnet-lab22` (`10.60.0.0/16`) with subnet `snet-backend` (`10.60.1.0/24`).

```bash
az group create --name rg-az104-lab22 --location eastus

az network vnet create \
  --resource-group rg-az104-lab22 \
  --name vnet-lab22 \
  --address-prefixes 10.60.0.0/16 \
  --subnet-name snet-backend \
  --subnet-prefixes 10.60.1.0/24
```

## Step 2 — Create a public Azure DNS zone

**Portal:** **+ Create a resource** → **DNS zone** → Name `contoso-az104-demo.com`, RG `rg-az104-lab22` → Create. (To be authoritative you would delegate your registrar's NS records to the Azure name servers shown on the Overview blade.)

```bash
az network dns zone create \
  --resource-group rg-az104-lab22 \
  --name contoso-az104-demo.com

# The name servers you'd delegate to at your registrar:
az network dns zone show -g rg-az104-lab22 -n contoso-az104-demo.com --query nameServers -o tsv
```

```powershell
New-AzDnsZone -ResourceGroupName rg-az104-lab22 -Name contoso-az104-demo.com
```

## Step 3 — Add A and CNAME records to the zone

**Portal:** DNS zone → **+ Record set** → Name `www`, Type **A**, TTL 3600, IP `20.1.2.3`. Add another: Name `app`, Type **CNAME**, alias `www.contoso-az104-demo.com`.

```bash
az network dns record-set a add-record \
  --resource-group rg-az104-lab22 \
  --zone-name contoso-az104-demo.com \
  --record-set-name www \
  --ipv4-address 20.1.2.3

az network dns record-set cname set-record \
  --resource-group rg-az104-lab22 \
  --zone-name contoso-az104-demo.com \
  --record-set-name app \
  --cname www.contoso-az104-demo.com
```

## Step 4 — Create a Standard public IP for the load balancer

**Portal:** Create Public IP `pip-lb`, SKU **Standard**, Static. Standard LB requires a Standard public IP.

```bash
az network public-ip create \
  --resource-group rg-az104-lab22 \
  --name pip-lb --sku Standard --allocation-method Static
```

## Step 5 — Create the public load balancer with a frontend

**Portal:** **+ Create a resource** → **Load balancer** → Name `lb-web`, Type **Public**, SKU **Standard**, Frontend IP `fe-web` using `pip-lb`.

```bash
az network lb create \
  --resource-group rg-az104-lab22 \
  --name lb-web \
  --sku Standard \
  --public-ip-address pip-lb \
  --frontend-ip-name fe-web \
  --backend-pool-name pool-web
```

```powershell
$pip = Get-AzPublicIpAddress -ResourceGroupName rg-az104-lab22 -Name pip-lb
$fe  = New-AzLoadBalancerFrontendIpConfig -Name fe-web -PublicIpAddress $pip
$pool = New-AzLoadBalancerBackendAddressPoolConfig -Name pool-web
New-AzLoadBalancer -ResourceGroupName rg-az104-lab22 -Name lb-web -Location eastus `
  -Sku Standard -FrontendIpConfiguration $fe -BackendAddressPool $pool
```

## Step 6 — Add a health probe

**Portal:** `lb-web` → **Health probes** → **+ Add** → Name `probe-http`, Protocol **TCP** (or HTTP path `/`), Port **80**, Interval 5s.

```bash
az network lb probe create \
  --resource-group rg-az104-lab22 \
  --lb-name lb-web \
  --name probe-http \
  --protocol Tcp --port 80 --interval 5
# The probe determines which backend instances are healthy enough to receive traffic.
```

## Step 7 — Add a load-balancing rule

**Portal:** `lb-web` → **Load balancing rules** → **+ Add** → Name `rule-http`, Frontend `fe-web`, Backend pool `pool-web`, Probe `probe-http`, Port 80 → Backend port 80.

```bash
az network lb rule create \
  --resource-group rg-az104-lab22 \
  --lb-name lb-web \
  --name rule-http \
  --protocol Tcp \
  --frontend-port 80 --backend-port 80 \
  --frontend-ip-name fe-web \
  --backend-pool-name pool-web \
  --probe-name probe-http \
  --idle-timeout 4 --enable-tcp-reset true
```

## Step 8 — Deploy two backend VMs and add them to the pool

**Portal:** Create `vm-web1` and `vm-web2` in `snet-backend` (no public IP), install a web server, then `lb-web` → **Backend pools** → `pool-web` → add both NICs.

```bash
for i in 1 2; do
  az vm create -g rg-az104-lab22 -n vm-web$i --image Ubuntu2204 --size Standard_B1s \
    --vnet-name vnet-lab22 --subnet snet-backend --public-ip-address "" \
    --admin-username azureuser --generate-ssh-keys \
    --custom-data "#cloud-config
runcmd:
  - apt-get update && apt-get install -y nginx
  - echo \"Served by \$(hostname)\" > /var/www/html/index.html"
done

# Add both NICs to the backend pool
for i in 1 2; do
  NIC=$(az vm show -g rg-az104-lab22 -n vm-web$i --query "networkProfile.networkInterfaces[0].id" -o tsv | xargs -I{} basename {})
  az network nic ip-config address-pool add \
    --resource-group rg-az104-lab22 --nic-name "$NIC" --ip-config-name ipconfig1 \
    --lb-name lb-web --address-pool pool-web
done
```

> **Note (Standard LB):** because these VMs have no public IP and the LB is Standard, add an outbound rule or NAT gateway if the VMs need internet egress (e.g. for `apt-get`). For the lab, a temporary public IP or a NAT gateway on the subnet works.

## Step 9 — Test load balancing

**Portal:** Copy the LB frontend IP from `pip-lb` Overview and browse to `http://<frontend-ip>` repeatedly — responses should alternate between `vm-web1` and `vm-web2`.

```bash
LBIP=$(az network public-ip show -g rg-az104-lab22 -n pip-lb --query ipAddress -o tsv)
for n in 1 2 3 4; do curl -s http://$LBIP; done
# You should see the two hostnames alternating as the rule distributes flows.
```

## Step 10 — Troubleshoot load balancing

**Portal:** `lb-web` → **Insights / Metrics** → check **Health Probe Status** and **Data Path Availability**. If a backend never receives traffic, verify: NSG allows the probe from `AzureLoadBalancer`, the app listens on the probe port, and the VM is in the pool.

```bash
# Confirm probe port is allowed and reachable, and inspect LB metrics
az monitor metrics list \
  --resource $(az network lb show -g rg-az104-lab22 -n lb-web --query id -o tsv) \
  --metric DipAvailability VipAvailability -o table
# DipAvailability = backend (probe) health; VipAvailability = frontend data path.

# From a peer VM, confirm the app answers on the probe port
az network watcher test-connectivity \
  --resource-group rg-az104-lab22 \
  --source-resource vm-web1 --dest-resource vm-web2 --dest-port 80
```

Common load-balancing faults to check:
- NSG on the subnet/NIC not allowing the `AzureLoadBalancer` service tag on the probe port.
- Health probe pointing at a port/path the app is not serving (backend shows unhealthy).
- Standard LB requires explicit outbound rules — VMs may have no internet egress by default.
- The VM NIC never added to the backend pool.

## Clean up

```bash
az group delete --name rg-az104-lab22 --yes --no-wait
```

> **Cost warning:** Standard **load balancers**, **public IPs**, gateways and Bastion bill **hourly** (plus data processed for Standard LB). DNS zones are billed per zone/month and per million queries. Delete this resource group promptly after the lab.

## References
- What is Azure DNS? — https://learn.microsoft.com/azure/dns/dns-overview
- What is Azure Load Balancer? — https://learn.microsoft.com/azure/load-balancer/load-balancer-overview
- Troubleshoot Azure Load Balancer — https://learn.microsoft.com/azure/load-balancer/load-balancer-troubleshoot
- Load Balancer health probes — https://learn.microsoft.com/azure/load-balancer/load-balancer-custom-probe-overview

## What you learned
- How to host a public DNS zone in Azure and create A and CNAME record sets.
- How registrar NS delegation makes an Azure DNS zone authoritative.
- How to assemble a Standard public load balancer: frontend IP, backend pool, health probe, and rule.
- Why the health probe (and its NSG allowance for the `AzureLoadBalancer` tag) is what decides traffic distribution.
- That Standard Load Balancer is secure-by-default and needs explicit outbound rules for backend internet egress.
- How to troubleshoot with DipAvailability/VipAvailability metrics and connectivity tests when traffic does not balance.
