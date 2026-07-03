# TGS-2023039182 — Microsoft Certified Azure Administrator Associate (AZ-104) Hands-On Labs

> **Course:** WSQ — Microsoft Certified Azure Administrator Associate (AZ-104)
> **Course Code:** TGS-2023039182
> **Register here:** https://www.tertiarycourses.com.sg/wsq-microsoft-certified-azure-administrator-associate-az-104.html

These are the official hands-on lab exercises for the WSQ Microsoft Certified Azure Administrator Associate (AZ-104) course delivered by [**Tertiary Infotech Academy Pte Ltd**](https://www.tertiarycourses.com.sg/).

A complete set of **26 step-by-step labs** aligned to the five official AZ-104 exam skill areas (skills measured as of April 17, 2026). Every lab runs in the **Azure Portal** (https://portal.azure.com) together with **Azure Cloud Shell** (https://shell.azure.com) using **Azure CLI** and **Azure PowerShell** — no local install required. An [Azure free account](https://azure.microsoft.com/free/) or an instructor-provided subscription is all you need.

---

## Courseware

The full WSQ courseware set for this course lives in [courseware/](courseware/). Every artifact is generated from a single source, so the slide deck, Lesson Plan and Learner Guide stay 100% aligned with the 26 labs.

| Artifact | File |
|----------|------|
| **Slide deck** (316 slides, AZ-104) | [courseware/Microsoft Azure Administrator (AZ-104)-v1.0.pptx](courseware/) · `.pdf` |
| **Lesson Plan** (daily schedule, slide-referenced) | [courseware/LP-Microsoft Azure Administrator (AZ-104).docx](courseware/) · `.pdf` |
| **Learner Guide** (all 26 labs) | [courseware/LG-Microsoft Azure Administrator (AZ-104).docx](courseware/) · `.pdf` |
| **Learner Guide (Markdown)** | [LG-Microsoft Azure Administrator (AZ-104).md](LG-Microsoft%20Azure%20Administrator%20(AZ-104).md) |

> **Note:** the confidential WSQ **assessment** materials (Written Assessment, Practical Performance and their answer keys) are intentionally **not** published to this repository.

---

## How to use

1. Sign in to the [Azure Portal](https://portal.azure.com) and open [Azure Cloud Shell](https://shell.azure.com) — the `az` CLI, `Az` PowerShell module, `azcopy`, and Bicep are pre-installed.
2. Pick a lab from the catalogue below and follow the steps in order. Each lab practises the Portal, Azure CLI, **and** Azure PowerShell — the three interfaces the exam expects.
3. Each lab is self-contained and creates its own resource group (for example `rg-az104-lab13`).
4. **Run the clean-up step at the end of every lab** to delete the resource group — Azure bills for running VMs, public IPs, gateways, and provisioned storage.
5. See [labs/README.md](labs/README.md) for the lab index with Azure services used per lab.

---

## Lab catalogue

### Domain 1 — Manage Azure identities and governance (20–25%)
- [Lab 1 — Manage Microsoft Entra Users and Groups](labs/lab-01-entra-users-groups.md)
- [Lab 2 — Manage External (B2B Guest) Users and Configure SSPR](labs/lab-02-external-users-sspr.md)
- [Lab 3 — Manage Access to Azure Resources with RBAC](labs/lab-03-rbac-roles.md)
- [Lab 4 — Implement and Manage Azure Policy](labs/lab-04-azure-policy.md)
- [Lab 5 — Manage Resource Groups, Locks and Tags](labs/lab-05-resource-groups-locks-tags.md)
- [Lab 6 — Manage Costs and Configure Management Groups](labs/lab-06-cost-management-groups.md)

### Domain 2 — Implement and manage storage (15–20%)
- [Lab 7 — Create and Configure Storage Accounts and Redundancy](labs/lab-07-storage-accounts-redundancy.md)
- [Lab 8 — Configure Access to Storage: Firewalls, SAS, Access Keys](labs/lab-08-storage-access-sas-firewall.md)
- [Lab 9 — Configure Azure Blob Storage: Tiers, Lifecycle, Versioning](labs/lab-09-blob-tiers-lifecycle.md)
- [Lab 10 — Configure Azure Files and Identity-Based Access](labs/lab-10-azure-files.md)
- [Lab 11 — Manage Data with Storage Explorer and AzCopy](labs/lab-11-storage-explorer-azcopy.md)

### Domain 3 — Deploy and manage Azure compute resources (20–25%)
- [Lab 12 — Automate Deployment with ARM Templates and Bicep](labs/lab-12-arm-bicep.md)
- [Lab 13 — Create a Virtual Machine, Encryption at Host, and Moving VMs](labs/lab-13-virtual-machines.md)
- [Lab 14 — VM Sizes, Disks, and Availability (Zones & Sets)](labs/lab-14-vm-disks-availability.md)
- [Lab 15 — Deploy and Configure a Virtual Machine Scale Set with Autoscale](labs/lab-15-vm-scale-sets.md)
- [Lab 16 — Containers: ACR, Container Instances, and Container Apps](labs/lab-16-containers.md)
- [Lab 17 — Azure App Service: Plans, Scaling, TLS, Custom Domains, Backup, Slots](labs/lab-17-app-service.md)

### Domain 4 — Implement and manage virtual networking (15–20%)
- [Lab 18 — Virtual Networks, Subnets & VNet Peering](labs/lab-18-vnet-peering.md)
- [Lab 19 — Public IPs, User-Defined Routes & Connectivity Troubleshooting](labs/lab-19-public-ip-udr.md)
- [Lab 20 — Network Security Groups, Application Security Groups & Effective Rules](labs/lab-20-nsg-asg.md)
- [Lab 21 — Azure Bastion, Service Endpoints & Private Endpoints](labs/lab-21-bastion-endpoints.md)
- [Lab 22 — Azure DNS & Load Balancing](labs/lab-22-dns-load-balancer.md)

### Domain 5 — Monitor and maintain Azure resources (10–15%)
- [Lab 23 — Azure Monitor: Metrics, Logs, KQL, and Alerts](labs/lab-23-azure-monitor.md)
- [Lab 24 — Azure Network Watcher and Connection Monitor](labs/lab-24-network-watcher.md)
- [Lab 25 — Azure Backup: Recovery Services Vault, Policies, Backup and Restore](labs/lab-25-azure-backup.md)
- [Lab 26 — Azure Site Recovery: Replicate, Fail Over, and Interpret Health](labs/lab-26-site-recovery.md)

---

## Prerequisites

- An Azure subscription (free account or instructor-provided) with at least **Contributor** access, and **Global Administrator** / **User Administrator** on the Microsoft Entra tenant for Domain 1 labs.
- A modern browser for the Azure Portal and Cloud Shell.
- Cloud Shell requires a small storage account on first launch — accept the default when prompted.

---

## Reference

- [labs/README.md](labs/README.md) — Lab index grouped by domain with Azure services used
- Official AZ-104 study guide: https://learn.microsoft.com/credentials/certifications/resources/study-guides/az-104
- Official Microsoft Learning labs: https://github.com/MicrosoftLearning/AZ-104-MicrosoftAzureAdministrator

---

## Important note

Always delete or deallocate resources when a lab is finished — Azure bills for running VMs, public IPs, gateways, and provisioned storage. Perform every lab only in a subscription you own or one your instructor has authorized. The clean-up section at the end of each lab removes the resource group created for that exercise.
