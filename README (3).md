# Lab 03: Modernizing to PaaS & Securing Secrets

**Author:** Dylan  
**Role:** Cloud Security Specialist  
**Video Walkthrough:** [▶ Watch on Loom](https://www.loom.com/share/0f179f2c054e4bdaafde9dddd70b894b)

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Naming Conventions](#naming-conventions)
- [Phase 1 — Decommission the Database VM](#phase-1--decommission-the-database-vm)
- [Phase 2 — Deploy Azure SQL Database (PaaS)](#phase-2--deploy-azure-sql-database-paas)
- [Phase 3 — Deploy Azure Key Vault](#phase-3--deploy-azure-key-vault)
- [Phase 4 — Store the SQL Password as a Secret](#phase-4--store-the-sql-password-as-a-secret)
- [Phase 5 — Enable Managed Identity & Grant Key Vault Access](#phase-5--enable-managed-identity--grant-key-vault-access)
- [Phase 6 — Validate with Azure Monitor](#phase-6--validate-with-azure-monitor)
- [Summary](#summary)
- [Next Steps](#next-steps)

---

## Overview

This lab refactors the two-tier infrastructure from **Lab 02** by replacing the self-managed database VM with **Azure SQL Database (PaaS)**, eliminating the operational overhead of managing a database server. Credentials are stored securely in **Azure Key Vault**, and the web VM is configured with a **Managed Identity** — removing all hardcoded passwords from the architecture.

This shift represents a real-world modernization pattern: moving from IaaS to PaaS while enforcing secrets management and identity-based access.

---

## Architecture

```
vm-web-01 (Managed Identity)
     │
     ▼
Azure Key Vault (kv-lab03-dylan)
     │  retrieves SqlAdminPassword
     ▼
Azure SQL Database (sqldb-app)
     │  hosted on
     └── sql-server-dylan
```

> No passwords are hardcoded anywhere in this architecture. The web VM authenticates via its system-assigned Managed Identity, retrieves the SQL credential from Key Vault at runtime, and connects to the managed database.

---

## Naming Conventions

| Resource | Name | Notes |
|----------|------|-------|
| Resource Group | `rg-lab03-dylan` | Scoped to this lab |
| SQL Server | `sql-server-dylan` | Must be globally unique |
| SQL Database | `sqldb-app` | Exact name required |
| Key Vault | `kv-lab03-dylan` | Must be globally unique |
| Region | Central US | Closest available US region |

---

## Phase 1 — Decommission the Database VM

The self-managed `vm-db-01` from Lab 02 is no longer needed. Remove it and its associated resources:

1. Open `rg-lab02-dylan` in the Azure Portal
2. Select and delete the following resources:
   - VM: `vm-db-01`
   - OS Disk: `vm-db-01-OsDisk`
   - Network Interface: `vm-db-01-nic`
3. Confirm deletion

> ⚠️ Do **not** delete the resource group or `vm-web-01` — both are required for the remainder of this lab.

---

## Phase 2 — Deploy Azure SQL Database (PaaS)

**Azure Portal → SQL Databases → + Create**

| Setting | Value |
|---------|-------|
| Resource Group | `rg-lab03-dylan` |
| Database Name | `sqldb-app` |
| Server | Create new → `sql-server-dylan` |
| Authentication | SQL authentication |
| Admin Login | `sqladmin` |
| Password | Strong password (saved to Key Vault in Phase 4) |
| Workload Environment | Development |
| Compute + Storage | DTU-based → Basic tier (~$4.99/month) |
| Networking | Public endpoint |
| Allow Azure Services | Yes |
| Add Current Client IP | Yes |
| Defender for SQL | Not now |
| Existing Data | None |

> **Cost Note:** The Basic DTU tier is selected here for lab purposes at ~$4.99/month. Review and confirm the tier before deploying to avoid unexpected charges.

✅ **Verify:** Database status shows **Online** in the Azure Portal.

---

## Phase 3 — Deploy Azure Key Vault

**Azure Portal → Key Vaults → + Create**

| Setting | Value |
|---------|-------|
| Resource Group | `rg-lab03-dylan` |
| Key Vault Name | `kv-lab03-dylan` |
| Region | Central US |
| Pricing Tier | Standard |
| Soft-Delete | Enabled |
| Purge Protection | Disabled |
| Access Model | Azure RBAC (default) |
| Networking | Public access — all networks |

✅ **Verify:** Vault URI is visible (e.g., `https://kv-lab03-dylan.vault.azure.net/`)

**Grant yourself Key Vault Administrator access:**

1. Navigate to Key Vault → **Access Control (IAM)**
2. Click **+ Add → Role Assignment**
3. Role: **Key Vault Administrator**
4. Assign to: your Azure account
5. Review + assign

> Without this role assignment, you will not be able to create or view secrets in the vault.

---

## Phase 4 — Store the SQL Password as a Secret

**Key Vault → Secrets → + Generate/Import**

| Field | Value |
|-------|-------|
| Name | `SqlAdminPassword` |
| Secret Value | SQL admin password set in Phase 2 |
| Enabled | Yes |

Click **Create**.

✅ **Verify:** `SqlAdminPassword` is listed in Secrets with status **Enabled**.

> **Security Note:** Storing credentials in Key Vault ensures they are never hardcoded in application configs, environment variables, or source code — a core principle of the [Azure Security Benchmark](https://learn.microsoft.com/en-us/security/benchmark/azure/).

---

## Phase 5 — Enable Managed Identity & Grant Key Vault Access

### Part A — Enable System-Assigned Managed Identity on the Web VM

1. Navigate to **Resource Groups → `rg-lab02-dylan` → `vm-web-01`**
2. Go to **Identity → System assigned**
3. Toggle status to **On** → Save

### Part B — Grant the VM Access to Key Vault Secrets

1. Navigate to **Key Vault `kv-lab03-dylan` → Access Control (IAM)**
2. Click **+ Add → Role Assignment**
3. Role: **Key Vault Secrets User**
4. Assign access to: **Managed Identity**
5. Select: `vm-web-01`
6. Review + assign

✅ **Verify:** IAM role assignments show `vm-web-01` listed under **Key Vault Secrets User**.

> The **Key Vault Secrets User** role grants read-only access to secret values — following the principle of least privilege. The VM can retrieve credentials at runtime without ever having write or admin access to the vault.

---

## Phase 6 — Validate with Azure Monitor

Confirm the database is live and observable:

1. Navigate to **SQL Databases → `sqldb-app` → Overview → Metrics**
2. Select metric: **DTU Percentage** (or CPU Percentage on Free tier)
3. Set aggregation: **Max**

✅ **Verify:** The chart displays activity, confirming the database is running and metrics are being collected.

---

## Summary

This lab demonstrates a complete modernization of a self-managed IaaS database to a fully managed PaaS model, with secrets management baked in:

- ✅ Decommissioned the database VM and migrated to Azure SQL Database (PaaS)
- ✅ Deployed Azure Key Vault and stored the SQL credential as a managed secret
- ✅ Enabled system-assigned Managed Identity on the web VM
- ✅ Granted least-privilege Key Vault access to the VM identity
- ✅ Validated database availability using Azure Monitor metrics
- ✅ Eliminated all hardcoded credentials from the architecture

---

## Next Steps

| Enhancement | Description |
|-------------|-------------|
| **Private Endpoint for SQL** | Remove the public SQL endpoint and route traffic privately through the VNet |
| **Private Endpoint for Key Vault** | Lock Key Vault access to the VNet, eliminating public exposure |
| **Key Vault Diagnostic Logs** | Enable logging to audit all secret access events |
| **App Configuration Service** | Centralize non-secret config values alongside Key Vault for a complete secrets strategy |
| **Azure Defender for SQL** | Enable threat detection and vulnerability assessments on the database |

---

*© Dylan — Cloud Security Specialist*
