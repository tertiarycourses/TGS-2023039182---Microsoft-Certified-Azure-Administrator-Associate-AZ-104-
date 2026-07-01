# Lab 3 — Manage Access to Azure Resources with RBAC

Maps to AZ-104 objective **Manage Azure identities and governance → Manage access to Azure resources** (interpret built-in Azure roles, assign roles at different scopes, interpret access assignments, create a custom role).

You will assign built-in roles at subscription, resource-group and resource scope, inspect effective access, then build and assign a custom role — the core of Azure role-based access control.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Azure RBAC (role assignments, role definitions), Resource Manager, a storage account as the target resource.

---

## Step 1 — Set variables and create a target resource group

**Portal:** **Resource groups → + Create**, Name `rg-az104-lab03`, pick a region, **Review + create**.

```bash
az login
SUB_ID=$(az account show --query id -o tsv)
LOCATION="southeastasia"
RG="rg-az104-lab03"

az group create --name "$RG" --location "$LOCATION"
echo "Subscription: $SUB_ID"
```

```powershell
$sub = (Get-AzContext).Subscription.Id
New-AzResourceGroup -Name "rg-az104-lab03" -Location "southeastasia"
```

## Step 2 — Create a storage account to act as a scoped resource

**Portal:** Inside `rg-az104-lab03` **+ Create → Storage account**, unique name, **Review + create**.

```bash
SA="stlab03$RANDOM"
az storage account create --name "$SA" --resource-group "$RG" \
  --location "$LOCATION" --sku Standard_LRS
SA_ID=$(az storage account show --name "$SA" --resource-group "$RG" --query id -o tsv)
echo "$SA_ID"
```

## Step 3 — Explore built-in role definitions

**Portal:** **Subscriptions → your subscription → Access control (IAM) → Roles** to browse built-in roles. Note the "big three": **Owner** (full access incl. granting access), **Contributor** (full access *except* granting access), **Reader** (view only). **User Access Administrator** manages access only.

```bash
# list built-in roles
az role definition list --query "[?roleType=='BuiltInRole'].roleName" -o tsv | sort | head -30

# inspect what Contributor can and cannot do (note NotActions removes Authorization/*)
az role definition list --name "Contributor" \
  --query "[0].permissions[0].{actions:actions, notActions:notActions}"

# inspect Reader
az role definition list --name "Reader" --query "[0].permissions[0].actions"
```

```powershell
Get-AzRoleDefinition -Name "Contributor" | Select-Object -ExpandProperty Actions
(Get-AzRoleDefinition -Name "Contributor").NotActions
```

> Exam insight: Contributor's `NotActions` includes `Microsoft.Authorization/*/Write` and `.../Delete` — that's why a Contributor cannot assign roles.

## Step 4 — Assign a role at subscription scope

**Portal:** **Subscriptions → Access control (IAM) → + Add → Add role assignment**, choose **Reader**, assign to the `grp-az104-admins` group (from Lab 1) or a user.

```bash
GRP_ID=$(az ad group show --group "grp-az104-admins" --query id -o tsv)

# assign Reader at subscription scope
az role assignment create \
  --assignee-object-id "$GRP_ID" \
  --assignee-principal-type Group \
  --role "Reader" \
  --scope "/subscriptions/$SUB_ID"
# this assigns the role at subscription scope; it is inherited by every RG and resource
```

```powershell
$g = Get-AzADGroup -DisplayName "grp-az104-admins"
New-AzRoleAssignment -ObjectId $g.Id -RoleDefinitionName "Reader" -Scope "/subscriptions/$sub"
```

## Step 5 — Assign a role at resource-group scope

**Portal:** **rg-az104-lab03 → Access control (IAM) → + Add → Add role assignment → Contributor**, assign to a user (e.g. Jane Doe).

```bash
JANE_ID=$(az ad user list --query "[?startsWith(displayName,'Jane')].id | [0]" -o tsv)

az role assignment create \
  --assignee-object-id "$JANE_ID" \
  --assignee-principal-type User \
  --role "Contributor" \
  --scope "/subscriptions/$SUB_ID/resourceGroups/$RG"
# Contributor here applies only to this RG and everything inside it
```

## Step 6 — Assign a role at resource scope

**Portal:** Open the **storage account → Access control (IAM) → + Add**, assign **Storage Blob Data Reader** to Raj Patel — this is data-plane access scoped to a single resource.

```bash
RAJ_ID=$(az ad user list --query "[?startsWith(displayName,'Raj')].id | [0]" -o tsv)

az role assignment create \
  --assignee-object-id "$RAJ_ID" \
  --assignee-principal-type User \
  --role "Storage Blob Data Reader" \
  --scope "$SA_ID"
# scoped to the storage account only — Raj can read blobs here and nowhere else
```

## Step 7 — Interpret access assignments

**Portal:** **rg-az104-lab03 → Access control (IAM) → Check access**, type a user's name to see their effective roles and where each is inherited from. Also **View my access** shows your own roles.

```bash
# all assignments at the storage account, including inherited ones
az role assignment list --scope "$SA_ID" --include-inherited -o table

# effective assignments for one user across the subscription
az role assignment list --assignee "$JANE_ID" --all -o table

# who has access at the RG (direct + inherited)
az role assignment list --resource-group "$RG" --include-inherited \
  --query "[].{principal:principalName, role:roleDefinitionName, scope:scope}" -o table
```

> Effective permission = the **union** of all assignments inherited from higher scopes plus those set directly. A **deny assignment** (rare, usually from Azure Blueprints/managed apps) overrides an allow.

## Step 8 — Create and assign a custom role

**Portal:** **Subscriptions → Access control (IAM) → + Add → Add custom role**, start from JSON, allow only restart of VMs, set assignable scope to the subscription, **Create**.

```bash
cat > /tmp/vm-restart-role.json <<EOF
{
  "Name": "Custom VM Restart Operator",
  "Description": "Can view and restart virtual machines only.",
  "Actions": [
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/instanceView/read"
  ],
  "NotActions": [],
  "AssignableScopes": [ "/subscriptions/$SUB_ID" ]
}
EOF

az role definition create --role-definition /tmp/vm-restart-role.json

# assign the custom role at the RG scope
az role assignment create \
  --assignee-object-id "$JANE_ID" --assignee-principal-type User \
  --role "Custom VM Restart Operator" \
  --scope "/subscriptions/$SUB_ID/resourceGroups/$RG"
```

```powershell
$role = Get-AzRoleDefinition "Contributor"
$role.Id = $null; $role.Name = "Custom VM Restart Operator"
$role.Description = "Can view and restart VMs only."
$role.Actions.Clear()
$role.Actions.Add("Microsoft.Compute/virtualMachines/read")
$role.Actions.Add("Microsoft.Compute/virtualMachines/restart/action")
$role.AssignableScopes.Clear(); $role.AssignableScopes.Add("/subscriptions/$sub")
New-AzRoleDefinition -Role $role
```

## Clean up

```bash
# remove role assignments (deleting the RG does not remove sub-scoped assignments)
az role assignment delete --assignee "$GRP_ID" --role "Reader" --scope "/subscriptions/$SUB_ID"
az role assignment delete --assignee "$JANE_ID" --role "Custom VM Restart Operator" \
  --scope "/subscriptions/$SUB_ID/resourceGroups/$RG"

# delete the custom role definition
az role definition delete --name "Custom VM Restart Operator"

# delete the resource group (removes the storage account and its resource-scoped assignments)
az group delete --name rg-az104-lab03 --yes --no-wait
```

## References

- Azure built-in roles — https://learn.microsoft.com/azure/role-based-access-control/built-in-roles
- Assign Azure roles using the Azure CLI — https://learn.microsoft.com/azure/role-based-access-control/role-assignments-cli
- Understand Azure role assignments and scope — https://learn.microsoft.com/azure/role-based-access-control/scope-overview
- Create or update Azure custom roles — https://learn.microsoft.com/azure/role-based-access-control/custom-roles

## What you learned

- Interpret the **Owner / Contributor / Reader / User Access Administrator** built-in roles and read their Actions/NotActions.
- Assign roles at **subscription, resource-group, and resource** scope and understand inheritance.
- Use **Check access** and `az role assignment list --include-inherited` to interpret effective access.
- Understand that effective permissions are the **union** of assignments and that **deny** wins over allow.
- Build a **custom role** with a minimal Actions set and a defined assignable scope, then assign it.
- Clean up assignments at each scope, remembering sub-scoped assignments survive RG deletion.
