# Lab 6 — Manage Costs and Configure Management Groups

Maps to AZ-104 objective **Manage Azure identities and governance → Manage subscriptions and governance** (analyze costs, create budgets and cost alerts, act on Azure Advisor recommendations, create and configure management groups).

You will explore Cost analysis, create a budget with alert thresholds, review Azure Advisor cost recommendations, and build a management-group hierarchy to organize subscriptions and apply governance at scale.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Microsoft Cost Management (Cost analysis, budgets, alerts), Azure Advisor, Management groups.

---

## Step 1 — Set variables

```bash
az login
SUB_ID=$(az account show --query id -o tsv)
TENANT_ID=$(az account show --query tenantId -o tsv)
echo "Subscription: $SUB_ID   Tenant: $TENANT_ID"
```

```powershell
$sub = (Get-AzContext).Subscription.Id
$tenant = (Get-AzContext).Tenant.Id
```

## Step 2 — Analyze costs with Cost analysis

**Portal (Cost analysis is Portal-driven for interactive views):** **Cost Management + Billing → Cost Management → Cost analysis**. Set the scope to your subscription, change the view to **Accumulated costs**, then **Group by → Service name** and **Group by → Resource group** to see where spend goes. Change the date range (This month / Last 30 days) and **Download** the data as CSV.

```bash
# query month-to-date actual cost by service via the Cost Management API
az costmanagement query \
  --type ActualCost \
  --scope "/subscriptions/$SUB_ID" \
  --timeframe MonthToDate \
  --dataset-aggregation '{ "totalCost": { "name": "Cost", "function": "Sum" } }' \
  --dataset-grouping name="ServiceName" type="Dimension" \
  -o table
```

## Step 3 — Create an action group for budget alerts (optional but realistic)

**Portal:** **Monitor → Alerts → Action groups → + Create**, add an **Email** action to your address so budget alerts can notify a group as well as the default contacts.

```bash
RG_MON="rg-az104-lab06"
az group create --name "$RG_MON" --location "southeastasia"

az monitor action-group create \
  --name "ag-budget-alerts" \
  --resource-group "$RG_MON" \
  --short-name "budgetAG" \
  --action email admin angch@tertiaryinfotech.com
```

## Step 4 — Create a budget with alert thresholds

**Portal:** **Cost Management → Budgets → + Add**. Scope = subscription, **Amount** = `100`, **Reset period** = Monthly, set **Alert conditions** at **50%**, **80%** and **100%** of budget, add alert recipients, **Create**.

```bash
# create a monthly subscription budget with two alert thresholds
az consumption budget create \
  --budget-name "budget-az104-monthly" \
  --amount 100 \
  --category Cost \
  --time-grain Monthly \
  --start-date "2026-07-01" \
  --end-date   "2027-06-30" \
  --notifications '{
    "warning80": {
        "enabled": true,
        "operator": "GreaterThanOrEqualTo",
        "threshold": 80,
        "contactEmails": ["angch@tertiaryinfotech.com"]
    },
    "critical100": {
        "enabled": true,
        "operator": "GreaterThanOrEqualTo",
        "threshold": 100,
        "contactEmails": ["angch@tertiaryinfotech.com"]
    }
  }'

az consumption budget list --query "[].{name:name, amount:amount, grain:timeGrain}" -o table
```

> Budgets are **notification** tools — they alert at % thresholds of *actual* or *forecast* spend; they do **not** stop resources from running. To act automatically, wire a budget alert to an Action Group that triggers automation.

## Step 5 — Set cost anomaly / spending-limit awareness

**Portal:** **Cost Management → Cost alerts** shows fired budget and anomaly alerts. Note that **Free/MSDN** subscriptions have a hard **spending limit** (auto-suspend at the credit cap), which is different from a budget alert.

```bash
# list any cost alerts that have fired
az rest --method get \
  --url "https://management.azure.com/subscriptions/$SUB_ID/providers/Microsoft.CostManagement/alerts?api-version=2023-08-01" \
  --query "value[].{name:name, type:properties.definition.type, status:properties.status}" -o table
```

## Step 6 — Review Azure Advisor cost recommendations

**Portal (Advisor is Portal-driven):** **Advisor → Overview** shows five pillars; open the **Cost** tab. Typical recommendations: *Right-size or shut down underutilized VMs*, *Buy reserved instances / savings plan*, *Delete unattached public IPs/disks*. Open a recommendation to see the action and estimated savings; you can **Postpone** or **Dismiss** it.

```bash
# pull Advisor recommendations, focused on the Cost category
az advisor recommendation list \
  --query "[?category=='Cost'].{impact:impact, problem:shortDescription.problem, resource:resourceMetadata.resourceId}" \
  -o table

# generate a fresh set of recommendations (async)
az advisor recommendation generate 2>/dev/null || true
```

```powershell
Get-AzAdvisorRecommendation -Category Cost |
  Select-Object Impact, @{n='Problem';e={$_.ShortDescription.Problem}}
```

## Step 7 — Create a management group hierarchy

**Portal:** **Management groups → + Create**. Create a top group `mg-az104-root` (child of the Tenant Root Group), then create `mg-az104-prod` and `mg-az104-nonprod` under it. Management groups let you apply RBAC and Policy above the subscription level.

```bash
# create the parent management group
az account management-group create \
  --name "mg-az104-root" \
  --display-name "AZ104 Root"

# create two children under it
az account management-group create \
  --name "mg-az104-prod" --display-name "AZ104 Prod" \
  --parent "mg-az104-root"

az account management-group create \
  --name "mg-az104-nonprod" --display-name "AZ104 NonProd" \
  --parent "mg-az104-root"

az account management-group list --query "[].{name:name, display:displayName}" -o table
```

```powershell
New-AzManagementGroup -GroupName "mg-az104-root" -DisplayName "AZ104 Root"
New-AzManagementGroup -GroupName "mg-az104-nonprod" -DisplayName "AZ104 NonProd" `
  -ParentId "/providers/Microsoft.Management/managementGroups/mg-az104-root"
```

> To enable management groups the first time, you must have the **Microsoft.Management** resource provider registered and the *Management Group Contributor* / Owner rights at the tenant root.

## Step 8 — Move a subscription and apply governance at the group

**Portal:** **Management groups → mg-az104-nonprod → + Add → Subscription**, pick your subscription. Then assign a Policy or role at `mg-az104-nonprod` and watch it flow down to the subscription.

```bash
# move the current subscription under the nonprod management group
az account management-group subscription add \
  --name "mg-az104-nonprod" \
  --subscription "$SUB_ID"

# assign a role at management-group scope (inherited by all child subs)
GRP_ID=$(az ad group show --group "grp-az104-admins" --query id -o tsv 2>/dev/null)
if [ -n "$GRP_ID" ]; then
  az role assignment create --assignee-object-id "$GRP_ID" --assignee-principal-type Group \
    --role "Reader" \
    --scope "/providers/Microsoft.Management/managementGroups/mg-az104-nonprod"
fi
```

> Policy and RBAC assignments made at a management group are **inherited** by every subscription and resource beneath it — the key reason to use management groups for enterprise governance.

## Clean up

```bash
# move the subscription back under the tenant root before deleting the groups
az account management-group subscription remove --name "mg-az104-nonprod" --subscription "$SUB_ID"

# remove role assignment if created
az role assignment delete --assignee "$GRP_ID" --role "Reader" \
  --scope "/providers/Microsoft.Management/managementGroups/mg-az104-nonprod" 2>/dev/null

# management groups must be empty of children/subscriptions before deletion
az account management-group delete --name "mg-az104-prod"
az account management-group delete --name "mg-az104-nonprod"
az account management-group delete --name "mg-az104-root"

# delete the budget, action group and monitoring RG
az consumption budget delete --budget-name "budget-az104-monthly"
az group delete --name rg-az104-lab06 --yes --no-wait
```

## References

- Explore and analyze costs with Cost analysis — https://learn.microsoft.com/azure/cost-management-billing/costs/quick-acm-cost-analysis
- Create and manage budgets — https://learn.microsoft.com/azure/cost-management-billing/costs/tutorial-acm-create-budgets
- Reduce service costs using Azure Advisor — https://learn.microsoft.com/azure/advisor/advisor-reference-cost-recommendations
- Create management groups for resource organization — https://learn.microsoft.com/azure/governance/management-groups/create-management-group-portal

## What you learned

- Use **Cost analysis** to break spend down by service and resource group, and query it via the Cost Management API.
- Create a **budget** with multiple **alert thresholds** and understand that budgets notify but do not stop spend.
- Distinguish a **budget alert** from a subscription **spending limit**.
- Review and act on **Azure Advisor** cost recommendations (right-size, reservations, orphaned resources).
- Build a **management-group hierarchy**, move a subscription into it, and apply **RBAC/Policy** that is inherited downward.
- Clean up management groups in order (empty of children/subscriptions before deletion).
