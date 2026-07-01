# Lab 1 — Manage Microsoft Entra Users and Groups

Maps to AZ-104 objective **Manage Azure identities and governance → Manage Microsoft Entra users and groups** (create users, create groups and manage membership, manage user and group properties, manage licenses in Microsoft Entra ID).

You will create cloud users, build a security group and a Microsoft 365 group, add members, edit user/group properties, and assign a license — the everyday identity-administration tasks the exam tests.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Microsoft Entra ID (users, groups, licenses).

---

## Step 1 — Sign in and confirm your tenant

**Portal:** Go to https://portal.azure.com, search for **Microsoft Entra ID** and open it. On the **Overview** blade note your **Primary domain** (for example `contoso.onmicrosoft.com`) — you'll use it as the suffix for every user's `userPrincipalName`.

```bash
# confirm you are signed in and see the tenant/default domain
az login
az account show --output table
az rest --method get \
  --url "https://graph.microsoft.com/v1.0/domains?\$select=id,isDefault" \
  --query "value[?isDefault].id" -o tsv
# the value returned is your primary/verified domain
```

```powershell
Connect-MgGraph -Scopes "User.ReadWrite.All","Group.ReadWrite.All","Organization.Read.All"
(Get-MgDomain | Where-Object IsDefault).Id
```

> Cloud Shell already has the Azure CLI and the `Microsoft.Graph` PowerShell module available. Set a variable for the domain so the commands below are copy-paste ready.

```bash
DOMAIN=$(az rest --method get \
  --url "https://graph.microsoft.com/v1.0/domains?\$select=id,isDefault" \
  --query "value[?isDefault].id" -o tsv)
echo "$DOMAIN"
```

## Step 2 — Create individual users

**Portal:** **Microsoft Entra ID → Users → All users → + New user → Create new user**. Set **User principal name** (e.g. `jdoe`), **Display name** `Jane Doe`, choose **Auto-generate password** or set one, then **Review + create**.

```bash
# create two cloud users. --force-change-password-next-sign-in makes them reset on first login
az ad user create \
  --display-name "Jane Doe" \
  --user-principal-name "jdoe@$DOMAIN" \
  --password "P@ssw0rd-Lab1!" \
  --force-change-password-next-sign-in true

az ad user create \
  --display-name "Raj Patel" \
  --user-principal-name "rpatel@$DOMAIN" \
  --password "P@ssw0rd-Lab1!" \
  --force-change-password-next-sign-in true
```

```powershell
$pw = @{ Password = "P@ssw0rd-Lab1!"; ForceChangePasswordNextSignIn = $true }
New-MgUser -DisplayName "Jane Doe" -UserPrincipalName "jdoe@$((Get-MgDomain | ? IsDefault).Id)" `
  -MailNickname "jdoe" -AccountEnabled -PasswordProfile $pw
```

## Step 3 — Edit user properties (job title, department, usage location)

**Portal:** Open **Users → Jane Doe → Edit properties**. On the **Job information** tab set **Job title** = `Cloud Administrator`, **Department** = `IT`. On **Settings**, set **Usage location** = your country (required before you can assign a license).

```bash
# update job info and usage location (usage location is required for licensing)
az ad user update --id "jdoe@$DOMAIN" \
  --set jobTitle="Cloud Administrator" department="IT" usageLocation="SG"
```

```powershell
Update-MgUser -UserId "jdoe@$((Get-MgDomain | ? IsDefault).Id)" `
  -JobTitle "Cloud Administrator" -Department "IT" -UsageLocation "SG"
```

> Set `usageLocation` to your own two-letter country code (e.g. `US`, `GB`, `IN`). Licenses cannot be assigned without it.

## Step 4 — Bulk create users (optional, Portal)

**Portal:** **Users → Bulk operations → Bulk create**. Download the CSV template, fill in `Name`, `User principal name`, `Initial password`, then upload and **Submit**. This is the exam-relevant way to onboard many users at once. Watch **Bulk operation results** for status.

```bash
# CLI equivalent: loop over a small list
for u in "Amy Tan:atan" "Ben Lee:blee"; do
  NAME="${u%%:*}"; ALIAS="${u##*:}"
  az ad user create --display-name "$NAME" \
    --user-principal-name "$ALIAS@$DOMAIN" \
    --password "P@ssw0rd-Lab1!" --force-change-password-next-sign-in true
done
```

## Step 5 — Create a security group and a Microsoft 365 group

**Portal:** **Microsoft Entra ID → Groups → + New group**. Create one group with **Group type = Security**, **Name** `grp-az104-admins`, **Membership type = Assigned**. Create a second with **Group type = Microsoft 365**, **Name** `grp-az104-project`.

```bash
# security group
az ad group create \
  --display-name "grp-az104-admins" \
  --mail-nickname "grp-az104-admins"

# Microsoft 365 group (Graph — CLI's ad group create makes security groups only)
az rest --method post \
  --url "https://graph.microsoft.com/v1.0/groups" \
  --headers "Content-Type=application/json" \
  --body '{
    "displayName": "grp-az104-project",
    "mailNickname": "grp-az104-project",
    "mailEnabled": true,
    "securityEnabled": false,
    "groupTypes": ["Unified"]
  }'
```

```powershell
New-MgGroup -DisplayName "grp-az104-admins" -MailNickname "grp-az104-admins" `
  -SecurityEnabled -MailEnabled:$false
New-MgGroup -DisplayName "grp-az104-project" -MailNickname "grp-az104-project" `
  -GroupTypes "Unified" -MailEnabled -SecurityEnabled:$false
```

## Step 6 — Add members to a group

**Portal:** **Groups → grp-az104-admins → Members → + Add members**, select Jane Doe and Raj Patel, then **Select**.

```bash
GROUP_ID=$(az ad group show --group "grp-az104-admins" --query id -o tsv)
JANE_ID=$(az ad user show --id "jdoe@$DOMAIN" --query id -o tsv)
RAJ_ID=$(az ad user show --id "rpatel@$DOMAIN" --query id -o tsv)

az ad group member add --group "$GROUP_ID" --member-id "$JANE_ID"
az ad group member add --group "$GROUP_ID" --member-id "$RAJ_ID"

# verify membership
az ad group member list --group "$GROUP_ID" --query "[].displayName" -o tsv
```

```powershell
$g = Get-MgGroup -Filter "displayName eq 'grp-az104-admins'"
$j = Get-MgUser  -Filter "userPrincipalName eq 'jdoe@$((Get-MgDomain | ? IsDefault).Id)'"
New-MgGroupMember -GroupId $g.Id -DirectoryObjectId $j.Id
```

## Step 7 — Edit group properties and configure a dynamic membership rule

**Portal:** Open **grp-az104-admins → Properties** to change description/owner. To try dynamic membership, create a new group with **Membership type = Dynamic User** and rule `(user.department -eq "IT")`. Dynamic groups require Microsoft Entra ID P1/P2.

```bash
# create a dynamic-membership security group via Graph
az rest --method post \
  --url "https://graph.microsoft.com/v1.0/groups" \
  --headers "Content-Type=application/json" \
  --body '{
    "displayName": "grp-az104-it-dynamic",
    "mailNickname": "grp-az104-it-dynamic",
    "mailEnabled": false,
    "securityEnabled": true,
    "groupTypes": ["DynamicMembership"],
    "membershipRule": "(user.department -eq \"IT\")",
    "membershipRuleProcessingState": "On"
  }'
# users whose department = IT (set in Step 3) are added automatically
```

## Step 8 — Assign a license

**Portal:** **Microsoft Entra ID → Billing → Licenses → All products** to see SKUs. Then **Users → Jane Doe → Licenses → + Assignments**, pick a product (e.g. **Microsoft Entra ID P2** or a trial), then **Save**. Group-based licensing: **Groups → grp-az104-admins → Licenses → + Assignments**.

```bash
# list the SKUs available in the tenant
az rest --method get \
  --url "https://graph.microsoft.com/v1.0/subscribedSkus?\$select=skuId,skuPartNumber,prepaidUnits" \
  --query "value[].{sku:skuPartNumber, id:skuId}" -o table

# assign a license to a user (replace <SKU_ID> with a skuId from above)
az rest --method post \
  --url "https://graph.microsoft.com/v1.0/users/jdoe@$DOMAIN/assignLicense" \
  --headers "Content-Type=application/json" \
  --body '{ "addLicenses": [ { "skuId": "<SKU_ID>", "disabledPlans": [] } ], "removeLicenses": [] }'
```

```powershell
Get-MgSubscribedSku | Select SkuPartNumber, SkuId, @{n='Consumed';e={$_.ConsumedUnits}}
Set-MgUserLicense -UserId "jdoe@$((Get-MgDomain | ? IsDefault).Id)" `
  -AddLicenses @(@{ SkuId = "<SKU_ID>" }) -RemoveLicenses @()
```

> If the tenant has no assignable licenses, note the click-path and skip the assignment; you can still demonstrate the process. **Group-based licensing** is preferred at scale because members inherit the license automatically.

## Clean up

Delete the test users and groups so the tenant stays tidy (there is no resource group for a pure-Entra lab).

```bash
# remove licenses first if you assigned any, then delete users and groups
az ad user delete --id "jdoe@$DOMAIN"
az ad user delete --id "rpatel@$DOMAIN"
az ad user delete --id "atan@$DOMAIN" 2>/dev/null
az ad user delete --id "blee@$DOMAIN" 2>/dev/null

az ad group delete --group "grp-az104-admins"
az ad group delete --group "grp-az104-project"
az ad group delete --group "grp-az104-it-dynamic" 2>/dev/null
```

> Deleted users/groups sit in **Deleted users / Deleted groups** for 30 days and can be restored. Permanently remove them from those blades if required.

## References

- Add or delete users — https://learn.microsoft.com/entra/fundamentals/how-to-create-delete-users
- Create a basic group and add members — https://learn.microsoft.com/entra/fundamentals/how-to-manage-groups
- Dynamic membership rules for groups — https://learn.microsoft.com/entra/identity/users/groups-dynamic-membership
- Assign or remove licenses — https://learn.microsoft.com/entra/fundamentals/license-users-groups

## What you learned

- Create cloud users individually, in bulk, and edit key properties (job title, department, usage location).
- Build both **security** and **Microsoft 365** groups and manage assigned membership.
- Configure a **dynamic membership** rule so group membership is driven by user attributes.
- Inspect tenant SKUs and assign licenses to a user, and understand why **group-based licensing** scales better.
- Perform the same tasks via Portal, Azure CLI, and Microsoft Graph PowerShell.
- Understand the 30-day soft-delete window for restoring users and groups.
