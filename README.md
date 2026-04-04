Watch Me Do the Lab HERE

https://www.loom.com/share/0f179f2c054e4bdaafde9dddd70b894b

# Lab 03: Modernizing to PaaS & Securing Secrets

**Author:** Dylan  
**Objective:** Refactor cloud infrastructure by replacing a self-managed database VM with Azure SQL Database (PaaS), store credentials securely in Azure Key Vault, and enable Managed Identity for the Web VM.

---
## Architecture Overview

- **Web VM:** vm-web-01 (from Lab 02)  
- **Key Vault:** kv-lab03-dylan  
- **SQL Database:** sqldb-app on sql-server-dylan  
- **Database VM:** vm-db-01 (DELETED)  

**Flow:** `vm-web-01` uses Managed Identity → retrieves SQL password from Key Vault → connects to Azure SQL Database (no hardcoded passwords).

---

## Naming Conventions

| Resource | Example | Notes |
|----------|--------|------|
| Resource Group | rg-lab03-dylan | Keep unique |
| SQL Server | sql-server-dylan | Globally unique |
| SQL Database | sqldb-app | Exact name required |
| Key Vault | kv-lab03-dylan | Globally unique |
| Azure Region | Central US | Closest available US region |

---

## Phase 1: Decommission Old Database VM

1. Open `rg-lab02-dylan`  
2. Select and delete:
   - VM: `vm-db-01`  
   - OS Disk: `vm-db-01-OsDisk`  
   - Network Interface: `vm-db-01-nic`  
3. Confirm deletion.  
⚠️ Do NOT delete the resource group or `vm-web-01`.

---

## Phase 2: Deploy Azure SQL Database (PaaS)

1. Search Azure Portal → SQL Databases → + Create → SQL Database  
2. Basics tab:
   - Subscription: Default  
   - Resource group: `rg-lab03-dylan`  
   - Database name: `sqldb-app`  
   - Server: Create new → `sql-server-dylan`  
   - Authentication: SQL authentication  
     - Admin login: `sqladmin`  
     - Password: strong password  
3. Workload environment: Development  
4. Compute + storage → DTU-based → Basic tier (~$5/month)  
5. Networking → Public endpoint  
   - Allow Azure services: Yes  
   - Add current client IP: Yes  
6. Security → Defender for SQL: Not now  
7. Additional settings → Use existing data: None  
8. Review + create → Verify tier = Basic, cost ≈ $4.99/month  

✅ Verify: Database status = Online

---

## Phase 3: Deploy Azure Key Vault

1. Search Azure Portal → Key Vaults → + Create  
2. Basics:
   - Resource group: `rg-lab03-dylan`  
   - Key vault name: `kv-lab03-dylan`  
   - Region: Central US  
   - Pricing tier: Standard  
   - Soft-delete: Enabled, Purge protection: Disabled  
3. Access Configuration: RBAC (default)  
4. Networking: Enable public access, Allow access from all networks  
5. Review + create → Confirm deployment  

✅ Verify: Vault URI visible (e.g., `https://kv-lab03-dylan.vault.azure.net/`)  

**Grant yourself access:**  
- Access control (IAM) → + Add → Role assignment → Key Vault Administrator → Your account

---

## Phase 4: Store the SQL Password as a Secret

1. Go to Key Vault → Secrets → + Generate/Import  
2. Fields:
   - Name: `SqlAdminPassword`  
   - Secret value: SQL admin password from Phase 2  
   - Enabled: Yes  
3. Click Create  

✅ Verify: Secret is listed and Enabled.

---

## Phase 5: Enable Managed Identity & Grant Key Vault Access

**Part A – Enable Managed Identity on VM:**  
1. Resource groups → `rg-lab02-dylan` → `vm-web-01` → Identity → System assigned → On → Save  

**Part B – Grant VM access to Key Vault:**  
1. Key Vault → Access control (IAM) → + Add → Role assignment → Key Vault Secrets User  
2. Assign access to: Managed Identity → Select `vm-web-01`  
3. Review + assign  

✅ Verify: Role assignment shows `vm-web-01` with Key Vault Secrets User

---

## Phase 6: Validate with Azure Monitor

1. SQL databases → `sqldb-app` → Overview → Metrics  
2. Metric: DTU percentage (or CPU percentage if on Free tier)  
3. Aggregation: Max  

✅ Verify: Chart displays activity confirming the database is live and observable.
