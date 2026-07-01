# Lab 2 — Manage External (B2B Guest) Users and Configure SSPR

Maps to AZ-104 objective **Manage Azure identities and governance → Manage external users and self-service password reset** (invite and manage B2B guest users, manage external collaboration settings, configure self-service password reset).

You will invite a guest user with Microsoft Entra B2B, review external collaboration settings, then enable and scope **self-service password reset (SSPR)** so users can reset their own passwords.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Microsoft Entra ID (External Identities / B2B, Password reset).

---

## Step 1 — Set your tenant variable

**Portal:** Open **Microsoft Entra ID → Overview** and note the **Primary domain**.

```bash
az login
DOMAIN=$(az rest --method get \
  --url "https://graph.microsoft.com/v1.0/domains?\$select=id,isDefault" \
  --query "value[?isDefault].id" -o tsv)
echo "$DOMAIN"
```

```powershell
Connect-MgGraph -Scopes "User.Invite.All","User.ReadWrite.All","Group.ReadWrite.All","Policy.ReadWrite.Authorization"
```

## Step 2 — Invite a B2B guest user

**Portal:** **Microsoft Entra ID → Users → All users → + New user → Invite external user**. Enter the guest's real email (use a mailbox you control, e.g. your personal Gmail), **Display name** `Contoso Partner`, add a personal message, then **Review + invite**. The guest receives an invitation email and redeems it to appear in your directory with `#EXT#` in the UPN and **User type = Guest**.

```bash
# send a B2B invitation; the invitedUserEmailAddress must be a real inbox you can open
az rest --method post \
  --url "https://graph.microsoft.com/v1.0/invitations" \
  --headers "Content-Type=application/json" \
  --body '{
    "invitedUserEmailAddress": "partner.test@gmail.com",
    "invitedUserDisplayName": "Contoso Partner",
    "inviteRedirectUrl": "https://myapps.microsoft.com",
    "sendInvitationMessage": true
  }'
# the response contains an inviteRedeemUrl and the created guest user object
```

```powershell
New-MgInvitation -InvitedUserEmailAddress "partner.test@gmail.com" `
  -InvitedUserDisplayName "Contoso Partner" `
  -InviteRedirectUrl "https://myapps.microsoft.com" `
  -SendInvitationMessage:$true
```

> A guest is created immediately in **PendingAcceptance** state. It becomes fully active once the invitee clicks the redemption link in the email.

## Step 3 — Verify the guest and add it to a group

**Portal:** **Users → filter User type = Guest** to confirm the guest exists. Open **Groups → + New group** (`grp-az104-partners`, Security, Assigned) and add the guest under **Members**.

```bash
# create a group and add the guest to it
az ad group create --display-name "grp-az104-partners" --mail-nickname "grp-az104-partners"

GUEST_ID=$(az ad user list \
  --query "[?userType=='Guest' && contains(displayName,'Contoso Partner')].id | [0]" -o tsv)
GRP_ID=$(az ad group show --group "grp-az104-partners" --query id -o tsv)

az ad group member add --group "$GRP_ID" --member-id "$GUEST_ID"
az ad group member list --group "$GRP_ID" --query "[].{name:displayName,type:userType}" -o table
```

## Step 4 — Review External collaboration settings

**Portal (this configuration is Portal-driven):** **Microsoft Entra ID → External Identities → External collaboration settings**. Review:
- **Guest user access restrictions** — how much of the directory guests can read.
- **Guest invite settings** — who may invite guests (recommended: *Member users and users assigned to specific admin roles*).
- **Collaboration restrictions** — an allow/deny list of domains guests may come from.

Set **Collaboration restrictions** to *Deny invitations to the specified domains* and add a test domain to see the effect, then revert to *Allow invitations to any domain*.

```bash
# read the current authorization policy that backs invite settings
az rest --method get \
  --url "https://graph.microsoft.com/v1.0/policies/authorizationPolicy" \
  --query "{allowInvitesFrom:allowInvitesFrom, guestUserRole:guestUserRoleId}"
```

> `allowInvitesFrom` values: `everyone`, `adminsAndGuestInviters`, `adminsGuestInvitersAndAllMembers`, `none`. The `guestUserRoleId` maps to the guest access level (restricted, default member-like, or same-as-member).

## Step 5 — Configure Cross-tenant access (B2B) defaults

**Portal:** **External Identities → Cross-tenant access settings → Default settings**. Review inbound/outbound B2B collaboration and trust settings. For a single-org lab leave defaults; note that **Organizational settings** let you add a specific partner tenant with custom inbound/outbound rules. This is where you control MFA/device-claim trust for guests.

## Step 6 — Enable Self-Service Password Reset (SSPR)

**Portal (SSPR is configured in the Portal):** **Microsoft Entra ID → Password reset → Properties**. Set **Self service password reset enabled** to **Selected** and pick a group (e.g. `grp-az104-admins`) or **All**, then **Save**. Scoping to **Selected** during rollout is the exam best practice.

> SSPR requires Microsoft Entra ID P1/P2 (or Microsoft 365 licensing that includes it) to enable for users. If your tenant has no license, follow the click-path and observe the blades.

## Step 7 — Configure SSPR authentication methods and registration

**Portal:** Still under **Password reset**:
- **Authentication methods** — set **Number of methods required to reset** = `2`, and enable **Mobile app notification/code**, **Email**, **Mobile phone (SMS)**, and **Security questions** as allowed methods.
- **Registration** — set **Require users to register when signing in** = **Yes** and **Number of days before re-confirm** = `180`.
- **Notifications** — enable **Notify users on password resets** and **Notify all admins when other admins reset**.

## Step 8 — Test the reset experience

**Portal / end-user:** In a private/incognito browser go to https://aka.ms/sspr (the password-reset portal) or https://aka.ms/ssprsetup (registration). Sign in as a scoped test user (e.g. Jane Doe from Lab 1), register the two methods, then use **Can't access your account?** on the sign-in page to walk the reset flow.

```bash
# admins can also force-reset a user's password (this is NOT SSPR, it's admin reset)
az ad user update --id "jdoe@$DOMAIN" \
  --password "N3wP@ssw0rd-Lab2!" --force-change-password-next-sign-in true
```

> Distinguish the two on the exam: **SSPR** = the *user* resets their own password after registering methods; **admin reset** = an administrator sets a new password for someone else.

## Clean up

```bash
# remove the guest, the partner group, and revert the test admin-reset user
az ad user delete --id "$GUEST_ID"
az ad group delete --group "grp-az104-partners"
```

Portal: set **Password reset → Properties → Self service password reset enabled** back to **None** if you enabled it only for testing, and set **External collaboration → Collaboration restrictions** back to *Allow invitations to any domain*.

## References

- Invite B2B collaboration users — https://learn.microsoft.com/entra/external-id/add-users-administrator
- External collaboration settings — https://learn.microsoft.com/entra/external-id/external-collaboration-settings-configure
- Enable Microsoft Entra self-service password reset — https://learn.microsoft.com/entra/identity/authentication/tutorial-enable-sspr
- How SSPR works — https://learn.microsoft.com/entra/identity/authentication/concept-sspr-howitworks

## What you learned

- Invite and manage **B2B guest users**, and recognize the `#EXT#` UPN and **Guest** user type.
- Control who can invite guests and from which domains via **External collaboration settings** and the authorization policy.
- Understand **cross-tenant access** defaults for inbound/outbound B2B collaboration and trust.
- Enable and **scope SSPR** to a group, and set the number of methods required to reset.
- Configure SSPR **authentication methods, registration, and notifications**.
- Tell the difference between **self-service reset** by a user and an **admin password reset**.
