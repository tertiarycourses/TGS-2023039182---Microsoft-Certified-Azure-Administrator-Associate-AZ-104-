# Microsoft Azure Administrator (AZ-104) — Lab Index

All labs run in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com), using **Azure CLI** and **Azure PowerShell**.

No installs are required on your own machine — Cloud Shell already has the `az` CLI, `Az` PowerShell module, `azcopy`, and Bicep pre-installed. Use an [Azure free account](https://azure.microsoft.com/free/) or an instructor-provided subscription. **Delete the resource group at the end of each lab** to avoid charges.

Skills measured are taken from the official AZ-104 study guide (as of April 17, 2026): https://learn.microsoft.com/credentials/certifications/resources/study-guides/az-104

---

## Domain 1 — Manage Azure identities and governance (20–25%)

| # | Lab | Azure services used |
|---|-----|---------------------|
| 1 | [Manage Microsoft Entra Users and Groups](lab-01-entra-users-groups.md) | Microsoft Entra ID, `az ad`, licenses |
| 2 | [Manage External Users and Self-Service Password Reset](lab-02-external-users-sspr.md) | Entra B2B guest users, SSPR |
| 3 | [Manage Access to Azure Resources with RBAC](lab-03-rbac-roles.md) | Azure RBAC, role assignments, scopes |
| 4 | [Implement and Manage Azure Policy](lab-04-azure-policy.md) | Azure Policy, initiatives, compliance |
| 5 | [Manage Subscriptions, Resource Groups, Locks, and Tags](lab-05-resource-groups-locks-tags.md) | Resource groups, resource locks, tags |
| 6 | [Manage Costs, Budgets, and Management Groups](lab-06-cost-management-groups.md) | Cost Management, budgets, Advisor, management groups |

## Domain 2 — Implement and manage storage (15–20%)

| # | Lab | Azure services used |
|---|-----|---------------------|
| 7 | [Create and Configure Storage Accounts and Redundancy](lab-07-storage-accounts-redundancy.md) | Storage accounts, LRS/ZRS/GRS, encryption, object replication |
| 8 | [Configure Access to Storage: Firewalls, SAS, Access Keys](lab-08-storage-access-sas-firewall.md) | Storage firewalls, SAS tokens, stored access policies, access keys |
| 9 | [Configure Azure Blob Storage: Tiers, Lifecycle, Versioning](lab-09-blob-tiers-lifecycle.md) | Blob containers, access tiers, soft delete, versioning, lifecycle |
| 10 | [Configure Azure Files and Identity-Based Access](lab-10-azure-files.md) | Azure Files, file shares, snapshots, identity-based access |
| 11 | [Manage Data with Storage Explorer and AzCopy](lab-11-storage-explorer-azcopy.md) | Azure Storage Explorer, AzCopy |

## Domain 3 — Deploy and manage Azure compute resources (20–25%)

| # | Lab | Azure services used |
|---|-----|---------------------|
| 12 | [Deploy Resources with ARM Templates and Bicep](lab-12-arm-bicep.md) | ARM templates, Bicep, template export |
| 13 | [Create and Configure Azure Virtual Machines](lab-13-virtual-machines.md) | Azure VMs, encryption at host, move VM |
| 14 | [Manage VM Disks, Sizes, Availability Zones and Sets](lab-14-vm-disks-availability.md) | Managed disks, VM sizes, availability zones/sets |
| 15 | [Deploy and Configure Virtual Machine Scale Sets](lab-15-vm-scale-sets.md) | VM Scale Sets, autoscale |
| 16 | [Provision Containers with ACR, ACI, and Container Apps](lab-16-containers.md) | Azure Container Registry, Container Instances, Container Apps |
| 17 | [Create and Configure Azure App Service](lab-17-app-service.md) | App Service plans, TLS, custom DNS, backup, deployment slots |

## Domain 4 — Implement and manage virtual networking (15–20%)

| # | Lab | Azure services used |
|---|-----|---------------------|
| 18 | [Configure Virtual Networks, Subnets, and Peering](lab-18-vnet-peering.md) | Virtual networks, subnets, VNet peering |
| 19 | [Public IPs, User-Defined Routes, and Connectivity Troubleshooting](lab-19-public-ip-udr.md) | Public IPs, route tables (UDR), connectivity |
| 20 | [Configure Network Security Groups and Application Security Groups](lab-20-nsg-asg.md) | NSGs, ASGs, effective security rules |
| 21 | [Implement Azure Bastion, Service Endpoints, and Private Endpoints](lab-21-bastion-endpoints.md) | Azure Bastion, service endpoints, private endpoints |
| 22 | [Configure Azure DNS and Load Balancing](lab-22-dns-load-balancer.md) | Azure DNS, public/internal load balancer |

## Domain 5 — Monitor and maintain Azure resources (10–15%)

| # | Lab | Azure services used |
|---|-----|---------------------|
| 23 | [Monitor Resources with Azure Monitor: Metrics, Logs, Alerts](lab-23-azure-monitor.md) | Azure Monitor, Log Analytics, KQL, alert rules, action groups |
| 24 | [Use Network Watcher and Connection Monitor](lab-24-network-watcher.md) | Network Watcher, IP flow verify, Connection Monitor |
| 25 | [Implement Azure Backup with a Recovery Services Vault](lab-25-azure-backup.md) | Recovery Services vault, backup policy, restore |
| 26 | [Configure Azure Site Recovery and Failover](lab-26-site-recovery.md) | Azure Site Recovery, replication, failover |

---

## Suggested order

Follow Domain 1 → 5 in numeric order. Each lab is self-contained and creates its own resource group named after the lab (for example `rg-az104-lab13`). **Run the clean-up step at the end of every lab** to delete that resource group before starting the next one.

## Prerequisites

- An Azure subscription (free account or instructor-provided) with at least **Contributor** access, and **Global Administrator** / **User Administrator** on the Microsoft Entra tenant for Domain 1 labs.
- A modern browser for the Azure Portal and Cloud Shell.
- Cloud Shell requires a small storage account on first launch — accept the default when prompted.
