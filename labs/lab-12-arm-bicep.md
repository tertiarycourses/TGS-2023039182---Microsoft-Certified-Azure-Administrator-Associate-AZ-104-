# Lab 12 — Automate Deployment with ARM Templates and Bicep

Maps to AZ-104 objective **Deploy and manage Azure compute resources → Automate deployment of resources by using ARM templates or Bicep files** (interpret an ARM/Bicep file, modify an existing ARM template, modify a Bicep file, deploy resources, export a deployment as an ARM template, convert an ARM template to Bicep).

In this lab you will read and understand an ARM template and a Bicep file, edit both, deploy a storage account with each, export a live deployment back to an ARM template, and decompile an ARM template into Bicep.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Azure Resource Manager (ARM), Bicep, Azure Storage, Azure Cloud Shell.

> Cost note: a Standard LRS storage account costs almost nothing when empty, but always run the **Clean up** step at the end.

---

## Step 1 — Create the resource group

**Portal:** Home → **Resource groups** → **+ Create** → Subscription → Name `rg-az104-lab12` → Region `East US` → **Review + create** → **Create**.

```bash
# Azure CLI
az group create --name rg-az104-lab12 --location eastus
```

```powershell
# Azure PowerShell
New-AzResourceGroup -Name rg-az104-lab12 -Location eastus
```

---

## Step 2 — Interpret an ARM template

Below is a minimal ARM template. Read it top-to-bottom to understand each section before deploying.

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": {
      "type": "string",
      "metadata": { "description": "Globally unique storage account name." }
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]"
    },
    "skuName": {
      "type": "string",
      "defaultValue": "Standard_LRS",
      "allowedValues": [ "Standard_LRS", "Standard_GRS" ]
    }
  },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2023-01-01",
      "name": "[parameters('storageAccountName')]",
      "location": "[parameters('location')]",
      "sku": { "name": "[parameters('skuName')]" },
      "kind": "StorageV2",
      "properties": { "supportsHttpsTrafficOnly": true }
    }
  ],
  "outputs": {
    "storageId": {
      "type": "string",
      "value": "[resourceId('Microsoft.Storage/storageAccounts', parameters('storageAccountName'))]"
    }
  }
}
```

Key sections to recognise on the exam:
- `parameters` — values supplied at deploy time (`storageAccountName`, `location`, `skuName`).
- `resources` — what gets created; `type`/`apiVersion` identify the resource provider version.
- `[resourceGroup().location]` and `[parameters(...)]` — ARM **template functions** and expressions.
- `outputs` — values returned after deployment.

Save it in Cloud Shell:

```bash
# In Azure Cloud Shell (Bash) create the file
cat > storage.json <<'EOF'
{ ... paste the JSON above ... }
EOF
```

---

## Step 3 — Modify the existing ARM template

Add a new allowed SKU (`Standard_ZRS`) and enforce a minimum TLS version. Edit `storage.json` so the `skuName` parameter and the resource `properties` become:

```json
"skuName": {
  "type": "string",
  "defaultValue": "Standard_LRS",
  "allowedValues": [ "Standard_LRS", "Standard_GRS", "Standard_ZRS" ]
}
```

```json
"properties": {
  "supportsHttpsTrafficOnly": true,
  "minimumTlsVersion": "TLS1_2",
  "allowBlobPublicAccess": false
}
```

# You just added a new allowed value and hardened the account with TLS 1.2 and no public blob access.

---

## Step 4 — Deploy the ARM template

**Portal:** Home → search **Deploy a custom template** → **Build your own template in the editor** → paste JSON → **Save** → fill parameters → **Review + create**.

```bash
# Azure CLI — storage account name must be globally unique, 3-24 lowercase alnum
SANAME="stlab12$RANDOM"
az deployment group create \
  --resource-group rg-az104-lab12 \
  --template-file storage.json \
  --parameters storageAccountName=$SANAME skuName=Standard_LRS

# Validate before deploying (dry run):
az deployment group validate \
  --resource-group rg-az104-lab12 \
  --template-file storage.json \
  --parameters storageAccountName=$SANAME
```

```powershell
# Azure PowerShell
$sa = "stlab12" + (Get-Random -Maximum 99999)
New-AzResourceGroupDeployment `
  -ResourceGroupName rg-az104-lab12 `
  -TemplateFile storage.json `
  -storageAccountName $sa -skuName Standard_LRS
```

---

## Step 5 — Interpret and write a Bicep file

Bicep is a cleaner DSL that compiles to ARM JSON. The same deployment in Bicep:

```bicep
@description('Globally unique storage account name.')
param storageAccountName string

param location string = resourceGroup().location

@allowed([ 'Standard_LRS', 'Standard_GRS', 'Standard_ZRS' ])
param skuName string = 'Standard_LRS'

resource sa 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: { name: skuName }
  kind: 'StorageV2'
  properties: {
    supportsHttpsTrafficOnly: true
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
  }
}

output storageId string = sa.id
```

Save it as `storage.bicep` in Cloud Shell.

---

## Step 6 — Modify the Bicep file and build it to ARM

Add a blob container as a child resource, then compile the Bicep to JSON to inspect the generated ARM.

```bicep
resource container 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-01-01' = {
  name: '${sa.name}/default/data'
}
```

```bash
# Compile Bicep -> ARM JSON (no deployment)
az bicep build --file storage.bicep
# Produces storage.json alongside it; open it to see the transpiled template
```

# az bicep build is the "compile" step; useful to review what ARM is actually generated.

---

## Step 7 — Deploy the Bicep file

**Portal:** **Deploy a custom template** also accepts `.bicep` files directly if you upload from your machine; otherwise deploy via CLI/PowerShell.

```bash
# Azure CLI deploys .bicep directly (Bicep CLI auto-installed)
SANAME2="stlab12b$RANDOM"
az deployment group create \
  --resource-group rg-az104-lab12 \
  --template-file storage.bicep \
  --parameters storageAccountName=$SANAME2 skuName=Standard_ZRS
```

```powershell
# Azure PowerShell also accepts .bicep
$sa2 = "stlab12b" + (Get-Random -Maximum 99999)
New-AzResourceGroupDeployment `
  -ResourceGroupName rg-az104-lab12 `
  -TemplateFile storage.bicep `
  -storageAccountName $sa2 -skuName Standard_ZRS
```

---

## Step 8 — Export a deployment as an ARM template

**Portal:** Open **rg-az104-lab12** → **Automation** section → **Export template** → review the generated JSON → **Download** or **Add to library**. You can also open a specific deployment: **Settings → Deployments →** select one → **Template**.

```bash
# Export the whole resource group as an ARM template
az group export --name rg-az104-lab12 > exported-rg.json

# Or export a single completed deployment's template
az deployment group export \
  --resource-group rg-az104-lab12 \
  --name storage \
  > exported-deployment.json
```

```powershell
Export-AzResourceGroup -ResourceGroupName rg-az104-lab12 -Path ./exported-rg.json
```

# "Export template" reverse-engineers existing resources into a reusable ARM template.

---

## Step 9 — Convert (decompile) an ARM template to Bicep

```bash
# Convert exported ARM JSON back into Bicep
az bicep decompile --file exported-rg.json
# Produces exported-rg.bicep — review and clean up any warnings/decompilation TODOs
```

# Decompile is a best-effort conversion; always review the output for correctness.

---

## Clean up

```bash
az group delete --name rg-az104-lab12 --yes --no-wait
```

## References
- Understand the structure and syntax of ARM templates: https://learn.microsoft.com/azure/azure-resource-manager/templates/syntax
- Bicep file structure and syntax: https://learn.microsoft.com/azure/azure-resource-manager/bicep/file
- Deploy resources with ARM templates and Azure CLI: https://learn.microsoft.com/azure/azure-resource-manager/templates/deploy-cli
- Decompile ARM template JSON to Bicep: https://learn.microsoft.com/azure/azure-resource-manager/bicep/decompile

## What you learned
- How to read the `parameters`, `resources`, `outputs`, and template-function expressions in an ARM template.
- How to modify an ARM template to add allowed values and harden a resource.
- How to author and edit a Bicep file, including child resources.
- How to deploy both ARM and Bicep files with `az deployment group create`.
- How to export existing resources with **Export template** / `az group export`.
- How to compile Bicep with `az bicep build` and convert ARM JSON to Bicep with `az bicep decompile`.
