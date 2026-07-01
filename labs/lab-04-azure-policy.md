# Lab 4 — Implement and Manage Azure Policy

Maps to AZ-104 objective **Manage Azure identities and governance → Implement and manage Azure Policy** (create and assign policy definitions and initiatives, interpret compliance, remediate non-compliant resources).

You will assign a built-in policy that restricts resource locations, build a custom definition, bundle policies into an initiative, read compliance results, and remediate non-compliant resources with a `DeployIfNotExists`/`Modify` effect.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Azure Policy (definitions, initiatives/set definitions, assignments, remediation), a managed identity for remediation, storage accounts as test resources.

---

## Step 1 — Set up variables and a scope

**Portal:** **Resource groups → + Create**, Name `rg-az104-lab04`.

```bash
az login
SUB_ID=$(az account show --query id -o tsv)
LOCATION="southeastasia"
RG="rg-az104-lab04"
SCOPE="/subscriptions/$SUB_ID/resourceGroups/$RG"

az group create --name "$RG" --location "$LOCATION"
```

```powershell
$sub = (Get-AzContext).Subscription.Id
New-AzResourceGroup -Name "rg-az104-lab04" -Location "southeastasia"
```

## Step 2 — Assign a built-in policy: allowed locations

**Portal:** **Policy → Assignments → Assign policy**. **Scope** = `rg-az104-lab04`; **Policy definition** = *Allowed locations*; on **Parameters** pick only `Southeast Asia`; **Review + create**.

```bash
# built-in "Allowed locations" definition id
LOC_DEF="e56962a6-4747-49cd-b67b-bf8b01975c4c"

az policy assignment create \
  --name "allowed-locations-lab04" \
  --display-name "Allowed locations (SEA only)" \
  --policy "$LOC_DEF" \
  --scope "$SCOPE" \
  --params '{ "listOfAllowedLocations": { "value": [ "southeastasia" ] } }'
```

```powershell
$def = Get-AzPolicyDefinition -Id "/providers/Microsoft.Authorization/policyDefinitions/e56962a6-4747-49cd-b67b-bf8b01975c4c"
New-AzPolicyAssignment -Name "allowed-locations-lab04" -PolicyDefinition $def `
  -Scope "/subscriptions/$sub/resourceGroups/rg-az104-lab04" `
  -PolicyParameterObject @{ listOfAllowedLocations = @("southeastasia") }
```

## Step 3 — Test the Deny effect

**Portal:** Try to create a resource in a disallowed region inside the RG — the deployment is blocked with a policy error.

```bash
# this should FAIL because eastus is not in the allowed list
az storage account create --name "stlab04deny$RANDOM" --resource-group "$RG" \
  --location "eastus" --sku Standard_LRS
# expected: "RequestDisallowedByPolicy" — Allowed locations denied it

# this should SUCCEED (allowed region)
az storage account create --name "stlab04ok$RANDOM" --resource-group "$RG" \
  --location "southeastasia" --sku Standard_LRS
```

> **Allowed locations** uses the **Deny** effect, so it blocks creation up front rather than just reporting non-compliance.

## Step 4 — Create a custom policy definition (require a tag)

**Portal:** **Policy → Definitions → + Policy definition**. Paste the rule, save at subscription scope.

```bash
cat > /tmp/require-env-tag.json <<'EOF'
{
  "if": {
    "allOf": [
      { "field": "type", "equals": "Microsoft.Storage/storageAccounts" },
      { "field": "tags['Environment']", "exists": "false" }
    ]
  },
  "then": { "effect": "audit" }
}
EOF

az policy definition create \
  --name "require-env-tag-on-storage" \
  --display-name "Storage accounts must have an Environment tag" \
  --mode Indexed \
  --rules /tmp/require-env-tag.json \
  --subscription "$SUB_ID"
```

## Step 5 — Create an initiative (policy set) and assign it

**Portal:** **Policy → Definitions → + Initiative definition**, add the custom tag policy plus the built-in *Allowed locations*, then **Save**. Assign it under **Assignments**.

```bash
CUSTOM_ID=$(az policy definition show --name "require-env-tag-on-storage" --query id -o tsv)

cat > /tmp/initiative.json <<EOF
[
  { "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/$LOC_DEF",
    "parameters": { "listOfAllowedLocations": { "value": [ "southeastasia" ] } } },
  { "policyDefinitionId": "$CUSTOM_ID" }
]
EOF

az policy set-definition create \
  --name "az104-governance-baseline" \
  --display-name "AZ-104 Governance Baseline" \
  --definitions /tmp/initiative.json \
  --subscription "$SUB_ID"

az policy assignment create \
  --name "governance-baseline-lab04" \
  --policy-set-definition "az104-governance-baseline" \
  --scope "$SCOPE"
```

## Step 6 — Interpret compliance

**Portal:** **Policy → Compliance**, open the assignment to see **Compliant / Non-compliant** counts and drill into the specific resources that failed.

```bash
# trigger an on-demand compliance scan for the subscription
az policy state trigger-scan --resource-group "$RG"

# read the latest compliance state (Compliant / NonCompliant per resource)
az policy state list --resource-group "$RG" \
  --query "[].{resource:resourceId, state:complianceState, policy:policyDefinitionName}" -o table

# summary counts
az policy state summarize --resource-group "$RG"
```

> Compliance evaluation runs automatically about every 24 hours and after each assignment change; `trigger-scan` forces it immediately.

## Step 7 — Remediate with a Modify/DeployIfNotExists policy

**Portal:** Assign a policy with a **Modify** or **DeployIfNotExists** effect (e.g. *Add or replace a tag on resources*). During assignment set **Create a Managed Identity**, region, and the role it needs. Then **Policy → Remediation → Create remediation task** for existing resources.

```bash
# built-in "Add a tag to resources" (Modify) definition
MODIFY_DEF="4f9dc7db-30c1-420c-b61a-e1d640128d26"

# assign with a system-assigned managed identity (required by Modify/DINE) and a location
az policy assignment create \
  --name "add-env-tag-lab04" \
  --policy "$MODIFY_DEF" \
  --scope "$SCOPE" \
  --params '{ "tagName": { "value": "Environment" }, "tagValue": { "value": "Lab" } }' \
  --mi-system-assigned \
  --identity-scope "$SCOPE" \
  --role "Contributor" \
  --location "$LOCATION"

# create a remediation task to fix already-existing resources
ASSIGN_ID=$(az policy assignment show --name "add-env-tag-lab04" --scope "$SCOPE" --query id -o tsv)
az policy remediation create \
  --name "remediate-env-tag" \
  --resource-group "$RG" \
  --policy-assignment "$ASSIGN_ID"

az policy remediation show --name "remediate-env-tag" --resource-group "$RG" \
  --query "{state:provisioningState, deployed:deploymentStatus}"
```

> **Modify** and **DeployIfNotExists** effects need a **managed identity** with the right role because they change/deploy resources. **Audit/Deny** effects do not.

## Step 8 — Add an exemption (optional)

**Portal:** **Policy → Assignments →** open the assignment **→ Exemptions → Create** to waive a specific resource for a reason/expiry.

```bash
SA_ID=$(az storage account list --resource-group "$RG" --query "[0].id" -o tsv)
az policy exemption create \
  --name "exempt-legacy-sa" \
  --policy-assignment "$ASSIGN_ID" \
  --exemption-category Waiver \
  --scope "$SA_ID" \
  --description "Legacy account exempt from Environment tag until migrated"
```

## Clean up

```bash
# remove exemption, remediation, assignments, initiative and custom definition
az policy exemption delete --name "exempt-legacy-sa" --scope "$SA_ID" 2>/dev/null
az policy remediation delete --name "remediate-env-tag" --resource-group "$RG" 2>/dev/null
az policy assignment delete --name "add-env-tag-lab04" --scope "$SCOPE"
az policy assignment delete --name "governance-baseline-lab04" --scope "$SCOPE"
az policy assignment delete --name "allowed-locations-lab04" --scope "$SCOPE"
az policy set-definition delete --name "az104-governance-baseline"
az policy definition delete --name "require-env-tag-on-storage"

az group delete --name rg-az104-lab04 --yes --no-wait
```

## References

- Overview of Azure Policy — https://learn.microsoft.com/azure/governance/policy/overview
- Create a custom policy definition — https://learn.microsoft.com/azure/governance/policy/tutorials/create-custom-policy-definition
- Remediate non-compliant resources — https://learn.microsoft.com/azure/governance/policy/how-to/remediate-resources
- Understand Azure Policy effects — https://learn.microsoft.com/azure/governance/policy/concepts/effect-basics

## What you learned

- Assign a **built-in policy** (Allowed locations) and observe the **Deny** effect blocking non-compliant deployments.
- Author a **custom policy definition** using the `if/then` rule structure.
- Bundle policies into an **initiative (policy set)** and assign it at a scope.
- Read **compliance state**, trigger on-demand scans, and drill into non-compliant resources.
- Use **Modify / DeployIfNotExists** effects with a **managed identity** and a **remediation task** to fix resources.
- Create a **policy exemption** to waive specific resources with a category and reason.
