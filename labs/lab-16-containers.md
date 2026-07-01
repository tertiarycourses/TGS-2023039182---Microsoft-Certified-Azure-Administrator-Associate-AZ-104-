# Lab 16 — Containers: ACR, Container Instances, and Container Apps

Maps to AZ-104 objective **Deploy and manage Azure compute resources → Create and configure containers** (create and manage an Azure Container Registry, provision a container with Azure Container Instances, provision a container with Azure Container Apps, and manage sizing and scaling).

In this lab you will build and push an image to Azure Container Registry, run it in Azure Container Instances, then deploy and scale it in Azure Container Apps.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Azure Container Registry (ACR), Azure Container Instances (ACI), Azure Container Apps, Log Analytics.

> Cost note: ACI and Container Apps bill per-second while running; a Basic ACR is inexpensive. Delete everything in **Clean up**.

---

## Step 1 — Create the resource group

**Portal:** **Resource groups → + Create** → `rg-az104-lab16` / `East US`.

```bash
az group create --name rg-az104-lab16 --location eastus
```

```powershell
New-AzResourceGroup -Name rg-az104-lab16 -Location eastus
```

---

## Step 2 — Create an Azure Container Registry

**Portal:** Search **Container registries → + Create** → RG `rg-az104-lab16` → Registry name `acrlab16<unique>` → SKU `Basic` → **Create**.

```bash
# Registry name must be globally unique, 5-50 alphanumeric, lowercase
ACR="acrlab16$RANDOM"
az acr create --resource-group rg-az104-lab16 --name $ACR --sku Basic --admin-enabled true
```

```powershell
$acr = "acrlab16" + (Get-Random -Maximum 99999)
New-AzContainerRegistry -ResourceGroupName rg-az104-lab16 -Name $acr -Sku Basic -EnableAdminUser
```

---

## Step 3 — Build and push an image with ACR Tasks

`az acr build` builds the image in the cloud and pushes it — no local Docker needed.

**Portal:** ACR → **Services → Tasks / Repositories** to view pushed images afterward.

```bash
# Build a sample image directly in ACR from a public quickstart repo
az acr build \
  --registry $ACR \
  --image webapp:v1 \
  https://github.com/Azure-Samples/acr-build-helloworld-node.git

# List repositories and tags
az acr repository list --name $ACR --output table
az acr repository show-tags --name $ACR --repository webapp --output table
```

# ACR Tasks perform the docker build/push server-side, which is ideal in Cloud Shell.

---

## Step 4 — Provision a container with Azure Container Instances (ACI)

**Portal:** Search **Container instances → + Create** → RG `rg-az104-lab16` → Name `aci-lab16` → Image source `Azure Container Registry` → select your registry/image → **Networking:** DNS label `acilab16<unique>`, port `80` → **Review + create**.

```bash
# Pull ACR credentials, then run the image in ACI with a public DNS name
ACR_SERVER=$(az acr show -n $ACR --query loginServer -o tsv)
ACR_USER=$(az acr credential show -n $ACR --query username -o tsv)
ACR_PASS=$(az acr credential show -n $ACR --query "passwords[0].value" -o tsv)

az container create \
  --resource-group rg-az104-lab16 \
  --name aci-lab16 \
  --image $ACR_SERVER/webapp:v1 \
  --registry-login-server $ACR_SERVER \
  --registry-username $ACR_USER \
  --registry-password $ACR_PASS \
  --dns-name-label acilab16$RANDOM \
  --ports 80 \
  --cpu 1 --memory 1.5

# Show the fully qualified domain name to browse to
az container show -g rg-az104-lab16 -n aci-lab16 --query ipAddress.fqdn -o tsv
```

```powershell
# New-AzContainerGroup -ResourceGroupName rg-az104-lab16 -Name aci-lab16 `
#   -Image "$acrServer/webapp:v1" -Cpu 1 -MemoryInGB 1.5 -Port 80 -DnsNameLabel acilab16
```

# --cpu and --memory control ACI sizing; ACI is best for single, short-lived or burst workloads.

---

## Step 5 — Create a Container Apps environment

Container Apps run on a managed environment backed by Log Analytics.

**Portal:** Search **Container Apps → + Create** → this also creates an **Environment**. Or create the environment first.

```bash
# Register the provider and install the extension (first time only)
az extension add --name containerapp --upgrade
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights

# Create the managed environment
az containerapp env create \
  --name aca-env-lab16 \
  --resource-group rg-az104-lab16 \
  --location eastus
```

---

## Step 6 — Provision a container with Azure Container Apps

**Portal:** **Container Apps → + Create** → RG `rg-az104-lab16` → App name `aca-lab16` → Environment `aca-env-lab16` → Image from your ACR → Ingress `Enabled`, target port `80`, accept traffic from anywhere → **Create**.

```bash
# Deploy from ACR with ingress enabled; --system-assigned lets it pull from ACR via managed identity
az containerapp create \
  --name aca-lab16 \
  --resource-group rg-az104-lab16 \
  --environment aca-env-lab16 \
  --image $ACR_SERVER/webapp:v1 \
  --registry-server $ACR_SERVER \
  --registry-username $ACR_USER \
  --registry-password $ACR_PASS \
  --target-port 80 \
  --ingress external \
  --cpu 0.5 --memory 1.0Gi

# Get the public URL
az containerapp show -g rg-az104-lab16 -n aca-lab16 --query properties.configuration.ingress.fqdn -o tsv
```

---

## Step 7 — Manage sizing and scaling of the Container App

Container Apps scale on replicas (including scale-to-zero) using HTTP or custom rules.

**Portal:** **aca-lab16 → Application → Scale and replicas** → set Min `0`, Max `5`; **+ Add scale rule** → HTTP scaling, concurrent requests `50`. **Containers → Edit and deploy** to change CPU/memory.

```bash
# Configure replica bounds and an HTTP concurrency scale rule
az containerapp update \
  --name aca-lab16 \
  --resource-group rg-az104-lab16 \
  --min-replicas 0 \
  --max-replicas 5 \
  --scale-rule-name http-rule \
  --scale-rule-type http \
  --scale-rule-http-concurrency 50

# Change CPU/memory sizing
az containerapp update -g rg-az104-lab16 -n aca-lab16 --cpu 1.0 --memory 2.0Gi
```

# Min replicas of 0 enables scale-to-zero so you pay nothing when idle; CPU and memory must match allowed combinations (e.g. 0.5 vCPU / 1.0Gi).

---

## Step 8 — Manage the registry (tags, cleanup of images)

```bash
# Delete an image tag from ACR to save space
az acr repository delete --name $ACR --image webapp:v1 --yes
# Show registry usage
az acr show-usage --name $ACR --output table
```

---

## Clean up

```bash
az group delete --name rg-az104-lab16 --yes --no-wait
```

## References
- Quickstart: Build and run a container image in ACR with ACR Tasks: https://learn.microsoft.com/azure/container-registry/container-registry-quickstart-task-cli
- Quickstart: Deploy a container instance with Azure CLI: https://learn.microsoft.com/azure/container-instances/container-instances-quickstart
- Quickstart: Deploy your first container app with the CLI: https://learn.microsoft.com/azure/container-apps/get-started
- Set scaling rules in Azure Container Apps: https://learn.microsoft.com/azure/container-apps/scale-app

## What you learned
- How to create an Azure Container Registry and build/push an image with `az acr build`.
- How to run a container in Azure Container Instances with `az container create`, including sizing via `--cpu`/`--memory`.
- How to create a Container Apps environment and deploy an app with `az containerapp create`.
- How to configure ingress and pull images from ACR.
- How to manage Container Apps sizing and autoscaling (min/max replicas, HTTP scale rules, scale-to-zero).
- When to choose ACI (burst/single tasks) versus Container Apps (scalable microservices).
