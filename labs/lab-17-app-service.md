# Lab 17 — Azure App Service: Plans, Scaling, TLS, Custom Domains, Backup, Networking, Slots

Maps to AZ-104 objective **Deploy and manage Azure compute resources → Create and configure Azure App Service** (provision an App Service plan, configure scaling, create an App Service, configure certificates and TLS, map a custom DNS name, configure backup, configure networking, configure deployment slots).

In this lab you will create an App Service plan and web app, configure scale-out rules, bind TLS/custom domains, enable backup, restrict networking, and use deployment slots for zero-downtime releases.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Azure App Service, App Service plans, Azure Monitor Autoscale, App Service Managed Certificates, Azure Storage (backup), VNet integration.

> Cost note: Most features here require **Standard (S1) or higher** plans, which bill hourly. Delete everything in **Clean up**.

---

## Step 1 — Create the resource group

**Portal:** **Resource groups → + Create** → `rg-az104-lab17` / `East US`.

```bash
az group create --name rg-az104-lab17 --location eastus
```

```powershell
New-AzResourceGroup -Name rg-az104-lab17 -Location eastus
```

---

## Step 2 — Provision an App Service plan

**Portal:** Search **App Service plans → + Create** → RG `rg-az104-lab17` → Name `plan-lab17` → OS `Linux` → Pricing plan `Standard S1` → **Create**.

```bash
# Standard tier is needed for autoscale, slots, custom domains, backup
az appservice plan create \
  --resource-group rg-az104-lab17 \
  --name plan-lab17 \
  --sku S1 \
  --is-linux
```

```powershell
New-AzAppServicePlan -ResourceGroupName rg-az104-lab17 -Name plan-lab17 `
  -Location eastus -Tier Standard -NumberofWorkers 1 -WorkerSize Small -Linux
```

---

## Step 3 — Create an App Service (web app)

**Portal:** **App Services → + Create → Web App** → RG `rg-az104-lab17` → Name `webapp-lab17<unique>` → Publish `Code` → Runtime `Node 20 LTS` → Plan `plan-lab17` → **Create**.

```bash
APP="webapp-lab17$RANDOM"
az webapp create \
  --resource-group rg-az104-lab17 \
  --plan plan-lab17 \
  --name $APP \
  --runtime "NODE:20-lts"

# Browse: https://<app>.azurewebsites.net
az webapp show -g rg-az104-lab17 -n $APP --query defaultHostName -o tsv
```

```powershell
$app = "webapp-lab17" + (Get-Random -Maximum 99999)
New-AzWebApp -ResourceGroupName rg-az104-lab17 -Name $app -AppServicePlan plan-lab17
```

---

## Step 4 — Configure scaling (scale up and scale out)

**Portal:** **plan-lab17 → Scale up (App Service plan)** to change tier; **Scale out (App Service plan)** → **Custom autoscale** → add a CPU rule.

```bash
# Scale UP (change the plan tier / instance size)
az appservice plan update --resource-group rg-az104-lab17 --name plan-lab17 --sku P1V3

# Scale OUT manually to 2 instances
az appservice plan update --resource-group rg-az104-lab17 --name plan-lab17 --number-of-workers 2

# Autoscale: add profile + CPU rule on the plan
PLANID=$(az appservice plan show -g rg-az104-lab17 -n plan-lab17 --query id -o tsv)
az monitor autoscale create -g rg-az104-lab17 --resource $PLANID \
  --name autoscale-lab17 --min-count 1 --max-count 5 --count 1
az monitor autoscale rule create -g rg-az104-lab17 --autoscale-name autoscale-lab17 \
  --condition "CpuPercentage > 70 avg 5m" --scale out 1
az monitor autoscale rule create -g rg-az104-lab17 --autoscale-name autoscale-lab17 \
  --condition "CpuPercentage < 30 avg 5m" --scale in 1
```

# "Scale up" changes the VM size/tier; "scale out" changes the number of instances.

---

## Step 5 — Map a custom DNS name

You need a domain you control. Add a CNAME (or A + TXT) at your DNS provider pointing to the app, then add the hostname.

**Portal:** **webapp-lab17 → Settings → Custom domains → + Add custom domain** → enter `www.contoso.com` → validate DNS → **Add**.

```bash
# After creating the CNAME record at your DNS provider:
az webapp config hostname add \
  --resource-group rg-az104-lab17 \
  --webapp-name $APP \
  --hostname www.contoso.com
```

```powershell
# Set-AzWebApp -ResourceGroupName rg-az104-lab17 -Name $app -HostNames @("www.contoso.com","$app.azurewebsites.net")
```

# Azure verifies domain ownership via a CNAME/TXT record before the binding is accepted.

---

## Step 6 — Configure certificates and TLS

**Portal:** **webapp-lab17 → Settings → Certificates → + Create App Service Managed Certificate** (free, for the custom domain) → then **Custom domains → Add binding → SNI SSL**. Set **Configuration → General settings → Minimum TLS version = 1.2** and **HTTPS Only = On**.

```bash
# Create a free managed certificate for the custom hostname
az webapp config ssl create \
  --resource-group rg-az104-lab17 \
  --name $APP \
  --hostname www.contoso.com

# Bind the cert (SNI). Get the thumbprint from the create output / ssl list
THUMB=$(az webapp config ssl list -g rg-az104-lab17 --query "[0].thumbprint" -o tsv)
az webapp config ssl bind \
  --resource-group rg-az104-lab17 \
  --name $APP \
  --certificate-thumbprint $THUMB \
  --ssl-type SNI

# Enforce HTTPS and TLS 1.2
az webapp update -g rg-az104-lab17 -n $APP --https-only true
az webapp config set -g rg-az104-lab17 -n $APP --min-tls-version 1.2
```

# Managed certificates are free and auto-renew but require the custom domain to be mapped first.

---

## Step 7 — Configure backup

Backup requires a Standard+ plan and a storage account + container.

**Portal:** **webapp-lab17 → Settings → Backups → Configure** → choose/create a storage account and container → set schedule (e.g. daily, retention 30 days) → **Save → Backup now**.

```bash
# Create a storage account and container for backups
STG="stlab17$RANDOM"
az storage account create -g rg-az104-lab17 -n $STG --sku Standard_LRS
az storage container create --account-name $STG --name backups

# Generate a SAS URL and configure scheduled backups
END=$(date -u -d "+1 year" '+%Y-%m-%dT%H:%MZ' 2>/dev/null || date -u -v+1y '+%Y-%m-%dT%H:%MZ')
SAS=$(az storage container generate-sas --account-name $STG --name backups \
  --permissions rwdl --expiry $END -o tsv)
CONTAINER_URL="https://$STG.blob.core.windows.net/backups?$SAS"

az webapp config backup update \
  --resource-group rg-az104-lab17 \
  --webapp-name $APP \
  --container-url "$CONTAINER_URL" \
  --frequency 1d \
  --retain-one true \
  --retention 30

# Trigger an on-demand backup
az webapp config backup create -g rg-az104-lab17 --webapp-name $APP --container-url "$CONTAINER_URL"
```

---

## Step 8 — Configure networking (VNet integration & access restrictions)

**Portal:** **webapp-lab17 → Settings → Networking** → **Outbound: VNet integration** to reach private resources; **Inbound: Access restrictions** to allow/deny by IP or service tag.

```bash
# Create a VNet + subnet delegated to App Service, then integrate
az network vnet create -g rg-az104-lab17 -n vnet-lab17 --address-prefix 10.0.0.0/16 \
  --subnet-name appsvc-subnet --subnet-prefix 10.0.1.0/24
az webapp vnet-integration add -g rg-az104-lab17 -n $APP \
  --vnet vnet-lab17 --subnet appsvc-subnet

# Inbound access restriction: allow only one IP, deny the rest
az webapp config access-restriction add -g rg-az104-lab17 -n $APP \
  --rule-name "allow-office" --action Allow --ip-address 203.0.113.10/32 --priority 100
```

# VNet integration is outbound (app reaching private resources); access restrictions are inbound firewall rules.

---

## Step 9 — Configure deployment slots

Slots let you deploy to a staging URL and swap into production with no downtime.

**Portal:** **webapp-lab17 → Deployment → Deployment slots → + Add slot** → Name `staging` → clone settings. Deploy to staging, test, then **Swap**.

```bash
# Create a staging slot
az webapp deployment slot create \
  --resource-group rg-az104-lab17 \
  --name $APP \
  --slot staging

# Swap staging into production
az webapp deployment slot swap \
  --resource-group rg-az104-lab17 \
  --name $APP \
  --slot staging \
  --target-slot production
```

```powershell
New-AzWebAppSlot -ResourceGroupName rg-az104-lab17 -Name $app -Slot staging
Switch-AzWebAppSlot -ResourceGroupName rg-az104-lab17 -Name $app -SourceSlotName staging -DestinationSlotName production
```

# Slots require Standard+; the swap warms up the target and applies slot-sticky settings.

---

## Clean up

```bash
az group delete --name rg-az104-lab17 --yes --no-wait
```

## References
- Manage an App Service plan: https://learn.microsoft.com/azure/app-service/app-service-plan-manage
- Scale up / scale out an app in Azure App Service: https://learn.microsoft.com/azure/app-service/manage-scale-up
- Add a TLS/SSL certificate and secure a custom domain: https://learn.microsoft.com/azure/app-service/configure-ssl-certificate
- Set up staging environments (deployment slots): https://learn.microsoft.com/azure/app-service/deploy-staging-slots

## What you learned
- How to provision an App Service plan and web app with `az appservice plan create` / `az webapp create`.
- The difference between scale up (tier/size) and scale out (instances), plus autoscale rules.
- How to map a custom domain with `az webapp config hostname add` and bind a free managed TLS certificate.
- How to enforce HTTPS-only and a minimum TLS version.
- How to configure scheduled backups to a storage account.
- How to configure outbound VNet integration and inbound access restrictions, and how to use deployment slots for zero-downtime swaps.
