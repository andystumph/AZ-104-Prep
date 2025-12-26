# Implement Backup and Recovery

> **Exam Weight:** Monitoring and Backup 10-15% (backup and disaster recovery are critical for business continuity)

## Table of Contents
- [Overview](#overview)
- [Recovery Services Vault](#recovery-services-vault)
- [Backup Vault](#backup-vault)
- [Azure VM Backup](#azure-vm-backup)
  - [Backup Policies](#backup-policies)
  - [Enable VM Backup](#enable-vm-backup)
  - [Restore VM](#restore-vm)
- [Azure Files Backup](#azure-files-backup)
- [SQL Server in Azure VM Backup](#sql-server-in-azure-vm-backup)
- [MARS Agent (On-Premises Backup)](#mars-agent-on-premises-backup)
- [Azure Site Recovery](#azure-site-recovery)
  - [Disaster Recovery Concepts](#disaster-recovery-concepts)
  - [Configure Site Recovery](#configure-site-recovery)
  - [Failover and Failback](#failover-and-failback)
- [Backup Security Features](#backup-security-features)
- [Best Practices](#best-practices)
- [PowerShell and CLI](#powershell-and-cli)
- [Exam Tips](#exam-tips)
- [Scenarios](#scenarios)
- [Troubleshooting](#troubleshooting)
- [Command Quick Reference](#command-quick-reference)

---

## Overview

**Azure Backup** provides secure, cost-effective backup solutions for Azure and on-premises resources. **Azure Site Recovery** (ASR) orchestrates disaster recovery with replication, failover, and failback capabilities.

**Key Services:**
- **Recovery Services Vault:** Stores backup data for VMs, SQL, files, on-premises workloads
- **Backup Vault:** Modern vault for Azure Disks, Blobs, PostgreSQL, AKS
- **Azure Site Recovery:** Disaster recovery with VM replication and failover

---

## Recovery Services Vault

**Purpose:** Central repository for backup data with RBAC, encryption, and long-term retention.

**Supported Workloads:**
- Azure VMs
- SQL Server in Azure VMs
- SAP HANA in Azure VMs
- Azure Files
- On-premises (via MARS agent, MABS, DPM)

**Key Properties:**
| Property | Details |
|----------|---------|
| **Storage Redundancy** | LRS, GRS (default), ZRS, RA-GRS |
| **Soft Delete** | 14-180 days retention after deletion (configurable) |
| **Cross-Region Restore** | Restore from secondary region (GRS/RA-GRS) |
| **Encryption** | Platform-managed keys (default) or customer-managed keys (CMK) |
| **Immutability** | Prevent deletion/modification of backups (locks vault) |

**Create Recovery Services Vault:**

**Portal:**
1. Search **Recovery Services vaults** → **Create**.
2. Select **Subscription**, **Resource group**, **Name**, **Region**.
3. Choose **Backup Storage Redundancy** (GRS recommended for DR).
4. Review and create.
5. Configure **Backup properties** (soft delete, cross-region restore, encryption).

**Limits:**
- 500 vaults per subscription per region
- 1000 Azure VMs per vault
- 50 MARS agents per vault
- 2000 datasources/items recommended per vault

---

## Backup Vault

**Purpose:** Modern backup solution for operational and snapshot-based backups.

**Supported Workloads:**
- Azure Disks
- Azure Blobs (operational backup)
- Azure Database for PostgreSQL
- Azure Kubernetes Service (AKS)

**Differences from Recovery Services Vault:**
- Operational-tier backups (continuous, no backup window)
- No agent required for most workloads
- Faster restore times (snapshots)
- Separate RBAC and pricing model

---

## Azure VM Backup

### Backup Policies

**Policy Components:**
1. **Backup Schedule:** Daily or weekly, specific time and timezone
2. **Instant Restore:** Snapshot retention (1-5 days, default 2)
3. **Retention:** Daily, weekly, monthly, yearly backup points

**Default Policy:**
- Daily backup at 10:00 PM
- 30 days retention
- 2 days instant restore snapshots

**Custom Policy Options:**
- **Enhanced Policy:** Multiple backups per day (hourly), selective disk backup, smart tiering to archive
- **Standard Policy:** Once-daily backup, full retention control

### Enable VM Backup

**Portal:**
1. Navigate to **VM** → **Backup**.
2. Select **Recovery Services vault** (create new or use existing).
3. Choose **Backup policy** (default or create custom).
4. Click **Enable backup**.
5. Azure installs **VMSnapshot extension** (Windows) or **VMSnapshotLinux** (Linux).

**What Happens:**
- Backup extension installed on VM agent
- First backup runs per schedule (or trigger on-demand)
- Snapshot taken (application-consistent if possible, else crash-consistent)
- Data transferred to vault (snapshot retained per instant restore setting)

**Application-Consistent vs Crash-Consistent:**
| Type | Description | Use Case |
|------|-------------|----------|
| **Application-Consistent** | Uses VSS (Windows) or pre/post scripts (Linux); captures in-flight transactions | Database VMs, transactional apps |
| **Crash-Consistent** | Disk snapshot without coordination; like VM crashed | VM powered off, VSS failure |
| **File-System Consistent** | Linux without pre/post scripts; file system metadata only | General Linux VMs |

### Restore VM

**Restore Options:**
1. **Create New VM:** Restore to new VM in same or different resource group/VNet
2. **Replace Existing:** Restore disks and replace existing VM's disks
3. **Restore Disks Only:** Download disks for manual attachment

**Portal:**
1. Navigate to **Recovery Services vault** → **Backup items** → **Azure Virtual Machine**.
2. Select **VM** → **Restore VM**.
3. Choose **Restore point** (latest, specific date, app-consistent).
4. Select **Restore configuration**:
   - **Create new:** VM name, resource group, VNet, subnet, staging storage account
   - **Replace existing:** Select VM to replace
   - **Restore disks:** Storage account for disk VHDs
5. Review and restore.
6. Monitor restore job in **Backup jobs**.

**Cross-Region Restore (CRR):**
- Available if vault uses GRS/RA-GRS
- Restore from secondary (paired) region during primary region outage
- Enable in vault **Backup Configuration** → **Cross Region Restore**

---

## Azure Files Backup

**Backup Method:** Snapshot-based (no data transferred to vault; snapshots stored with storage account).

**Enable Backup:**
1. Navigate to **Storage account** → **File shares** → Select file share → **Backup**.
2. Select **Recovery Services vault** (or create new).
3. Choose **Backup policy** (schedule and retention).
4. Click **Enable backup**.

**Restore Options:**
- Full share restore (to original or alternate storage account)
- Item-level restore (specific files/folders)

**Limitations:**
- Max 200 file shares per storage account with backup
- Snapshots count toward storage account limit (200 per file share)

---

## SQL Server in Azure VM Backup

**Features:**
- **Full, Differential, Log backups:** Granular recovery (point-in-time)
- **Auto-protection:** Automatically backup new databases
- **Backup compression:** Reduce backup size (configurable)
- **Long-term retention:** Up to 10 years

**Enable SQL Backup:**
1. Navigate to **Recovery Services vault** → **Backup** → **SQL in Azure VM**.
2. **Discover DBs:** Select VM; Azure discovers SQL instances and databases.
3. **Install extension:** Azure installs **AzureBackupWindowsWorkload** extension.
4. **Configure backup:** Select databases, choose policy (or create new).
5. **Enable backup**.

**Backup Policy for SQL:**
- **Full Backup:** Daily or weekly
- **Differential Backup:** Daily (if full is weekly)
- **Log Backup:** Every 15 minutes to 24 hours
- **Retention:** Daily (7+ days), weekly, monthly, yearly

**Restore Options:**
- Restore as new database (same or alternate VM)
- Overwrite existing database
- Restore to point-in-time (via log chain)
- Restore individual files (for full backups)

---

## MARS Agent (On-Premises Backup)

**MARS (Microsoft Azure Recovery Services) Agent:** Backs up files, folders, and system state from on-premises Windows machines to Azure.

**Install MARS Agent:**
1. Download agent from **Recovery Services vault** → **Backup** → **On-premises**.
2. Run **MARSAgentInstaller.exe** on Windows machine.
3. Download **Vault credentials** from portal.
4. Register server with vault using credentials.
5. Set **Encryption passphrase** (store securely; required for restore).

**Configure Backup:**
1. Launch **Microsoft Azure Backup** console on server.
2. Click **Schedule Backup** → Select files/folders.
3. Set **Backup schedule** (up to 3 times per day).
4. Define **Retention policy** (daily, weekly, monthly, yearly).
5. Choose **Initial backup type** (online or offline via Azure Import/Export).
6. Complete wizard; backup runs per schedule.

**Restore Files:**
1. Open **Microsoft Azure Backup** console → **Recover Data**.
2. Choose restore location (this server or another server).
3. Select **Recovery point** (browse by date).
4. Choose files/folders to restore.
5. Specify **Recovery options** (original location, alternate, overwrite settings).
6. Enter **Encryption passphrase**.
7. Start recovery.

**Cross-Region Restore (CRR) with MARS:**
- Download **secondary region vault credentials** from portal
- Use credentials in restore wizard to recover from paired region

---

## Azure Site Recovery

### Disaster Recovery Concepts

| Concept | Definition |
|---------|------------|
| **RPO (Recovery Point Objective)** | Maximum acceptable data loss (time between backups) |
| **RTO (Recovery Time Objective)** | Maximum acceptable downtime (time to restore service) |
| **Replication** | Continuous copying of VM data to target region/site |
| **Failover** | Switch workloads from primary to secondary site |
| **Failback** | Return workloads to primary site after recovery |
| **Test Failover** | Non-disruptive DR drill (isolated VNet) |

**ASR Capabilities:**
- **Azure to Azure:** Replicate Azure VMs between regions
- **VMware/Physical to Azure:** Replicate on-premises VMware VMs or physical servers
- **Hyper-V to Azure:** Replicate on-premises Hyper-V VMs
- **RTO SLA:** ~1 hour (actual RTO depends on workload)
- **RPO:** As low as 30 seconds (Hyper-V); continuous for Azure VMs and VMware

### Configure Site Recovery

**Azure to Azure DR:**

**Prerequisites:**
- VMs in source region
- Recovery Services vault in target region (different from source)
- VNet in target region

**Enable Replication:**
1. Navigate to **Recovery Services vault** → **Site Recovery** → **Replicate**.
2. Select **Source:** Azure, source region, resource group, deployment model.
3. Select **Target:** Target region (paired region recommended).
4. Configure **Replication settings**:
   - Target resource group, VNet, availability options
   - Replication policy (app-consistent snapshot frequency: 1-12 hours; crash-consistent: 5 minutes)
5. Select **VMs** to replicate.
6. Review and **Enable replication**.

**Replication Process:**
1. ASR installs **Mobility Service** extension on VMs.
2. Initial replication copies all VM data to target region (cache storage account).
3. Ongoing replication sends delta changes (continuous).
4. Recovery points created per policy (crash-consistent every 5 min, app-consistent per schedule).

### Failover and Failback

**Test Failover (DR Drill):**
1. Navigate to **Replicated items** → Select VM → **Test Failover**.
2. Choose **Recovery point** (latest, latest app-consistent, custom).
3. Select **Azure virtual network** (isolated test network, not production).
4. Click **OK**; Azure creates test VM in target region.
5. Validate application functionality.
6. **Cleanup test failover** when complete (deletes test VM).

**Planned Failover (No Data Loss):**
- For scheduled maintenance or when primary site is still accessible
- Shuts down source VMs before failover (ensures zero data loss)
- Used for failback from Azure to on-premises

**Unplanned Failover (Disaster):**
1. Navigate to **Replicated items** → Select VM → **Failover**.
2. Choose **Recovery point**:
   - **Latest (lowest RPO):** Most recent data (processes any pending data)
   - **Latest processed:** Last processed recovery point (low RTO, may lose recent changes)
   - **Latest app-consistent:** Last app-consistent snapshot (higher RPO, cleaner state)
   - **Custom:** Specific point-in-time
3. Check **Shut down machine before beginning failover** (if source accessible).
4. Click **OK**; Azure fails over to target region.
5. **Commit failover** after validation (deletes other recovery points).

**Reprotect (Prepare for Failback):**
1. After failover, VMs run in target region.
2. Navigate to VM → **Re-protect**.
3. Configure reverse replication (target → source).
4. Azure starts replicating back to source region.

**Failback:**
1. When source region is ready, perform **Planned Failover** from target to source.
2. Choose failback recovery point (latest recommended).
3. VMs fail back to source region.
4. **Re-protect again** to resume DR replication (source → target).

**Recovery Plans:**
- Group multiple VMs for coordinated failover
- Add pre/post scripts (Azure Automation runbooks)
- Define boot order and delays between tiers
- Useful for multi-tier applications (e.g., SQL → App → Web)

---

## Backup Security Features

### Soft Delete

**Purpose:** Protect against accidental or malicious deletion.

**How It Works:**
- Deleted backup data retained for 14-180 days (configurable)
- Soft-deleted data doesn't count toward billing after 14 days
- Can undelete during retention period

**Enable Soft Delete:**
1. Navigate to **Recovery Services vault** → **Properties** → **Security Settings**.
2. Click **Update**.
3. Enable **Soft delete** for cloud workloads.
4. Set **Retention period** (14-180 days).
5. Optionally enable **Always-on soft delete** (cannot be disabled; permanent setting).
6. Click **Update**.

**Recover Soft-Deleted Item:**
1. Navigate to **Backup items** → Filter by **Soft Deleted**.
2. Select item → **Undelete**.
3. Item restored to active protection.

### Immutable Vaults

**Purpose:** Prevent deletion or modification of backups (WORM - Write Once Read Many).

**Use Cases:**
- Regulatory compliance (SOX, HIPAA, GDPR)
- Ransomware protection (prevents malicious deletion)
- Legal hold requirements

**Enable Immutability:**
1. Navigate to **Recovery Services vault** → **Properties** → **Immutable vault**.
2. Click **Enable**.
3. Choose **Locked** or **Unlocked**:
   - **Unlocked:** Can be disabled (for testing)
   - **Locked:** Irreversible; immutability cannot be removed
4. Confirm enablement.

**Impacts:**
- Cannot reduce retention or delete backups until retention expires
- Cannot delete vault until all backups expire
- Cannot disable backup on protected items (must wait for retention expiry)

### Encryption

**At Rest:**
- **Platform-Managed Keys (PMK):** Default; Azure manages encryption keys
- **Customer-Managed Keys (CMK):** Store keys in Azure Key Vault; full control over key lifecycle

**In Transit:**
- HTTPS/TLS for data transfer to Azure
- MARS agent: Passphrase-encrypted before upload (user-defined passphrase)

**Enable CMK:**
1. Navigate to **Recovery Services vault** → **Properties** → **Encryption**.
2. Select **Use your own key**.
3. Choose **Key Vault** and **Key**.
4. Assign managed identity permissions to Key Vault.
5. Update encryption settings.

---

## Best Practices

- **Vault Placement:** Create vault in same region as resources; use GRS for cross-region DR.
- **Backup Policies:** Use default policy for quick setup; customize for production (retention, frequency).
- **Instant Restore:** 2-5 days recommended (faster restore, higher snapshot storage cost).
- **Test Restores:** Regularly validate backups (quarterly test failovers for ASR).
- **Soft Delete:** Always enable (14-day minimum); use 90+ days for compliance.
- **Immutability:** Enable for critical workloads; lock only after thorough testing.
- **MARS Passphrase:** Store securely offline (cannot recover data without it).
- **ASR:** Use recovery plans for multi-VM apps; automate with runbooks; test failover quarterly.
- **Monitoring:** Configure alerts for failed backups, expired policies, replication health.
- **Cost Optimization:** Use LRS for non-critical workloads; archive tier for long-term retention (SQL/VM).

---

## PowerShell and CLI

```powershell
# Create Recovery Services Vault
$rg = "backup-rg"
$loc = "eastus"
$vaultName = "prodVault"

New-AzRecoveryServicesVault -ResourceGroupName $rg -Name $vaultName -Location $loc
$vault = Get-AzRecoveryServicesVault -Name $vaultName -ResourceGroupName $rg
Set-AzRecoveryServicesBackupProperty -Vault $vault -BackupStorageRedundancy GeoRedundant

# Enable VM Backup
$pol = Get-AzRecoveryServicesBackupProtectionPolicy -Name "DefaultPolicy" -VaultId $vault.ID
Enable-AzRecoveryServicesBackupProtection -ResourceGroupName "vm-rg" -Name "myVM" -Policy $pol -VaultId $vault.ID

# Trigger On-Demand Backup
$container = Get-AzRecoveryServicesBackupContainer -ContainerType AzureVM -FriendlyName "myVM" -VaultId $vault.ID
$item = Get-AzRecoveryServicesBackupItem -Container $container -WorkloadType AzureVM -VaultId $vault.ID
Backup-AzRecoveryServicesBackupItem -Item $item -VaultId $vault.ID

# Restore VM to New VM
$startDate = (Get-Date).AddDays(-7)
$endDate = Get-Date
$rp = Get-AzRecoveryServicesBackupRecoveryPoint -Item $item -StartDate $startDate -EndDate $endDate -VaultId $vault.ID
$restoreJob = Restore-AzRecoveryServicesBackupItem -RecoveryPoint $rp[0] -StorageAccountName "restoresa" -StorageAccountResourceGroupName $rg -TargetResourceGroupName "restore-rg" -VaultId $vault.ID -VaultLocation $vault.Location

# Monitor Restore Job
Get-AzRecoveryServicesBackupJob -Job $restoreJob -VaultId $vault.ID

# Enable Soft Delete
Set-AzRecoveryServicesVaultProperty -Vault $vault -SoftDeleteFeatureState Enable -SoftDeleteRetentionInDays 30

# Enable SQL Backup (requires VM with SQL)
$targetVault = Get-AzRecoveryServicesVault -Name "sqlVault" -ResourceGroupName $rg
$container = Get-AzRecoveryServicesBackupContainer -ContainerType AzureVMAppContainer -FriendlyName "sqlVM" -VaultId $targetVault.ID
$item = Get-AzRecoveryServicesBackupProtectableItem -WorkloadType MSSQL -Container $container -VaultId $targetVault.ID
$pol = Get-AzRecoveryServicesBackupProtectionPolicy -Name "HourlyLogBackup" -VaultId $targetVault.ID
Enable-AzRecoveryServicesBackupProtection -ProtectableItem $item -Policy $pol -VaultId $targetVault.ID

# Azure Site Recovery: Enable Replication
$vault = Get-AzRecoveryServicesVault -Name "asrVault" -ResourceGroupName $rg
Set-AzRecoveryServicesAsrVaultContext -Vault $vault
$primaryFabric = Get-AzRecoveryServicesAsrFabric -Name "asr-primary-fabric"
$protectionContainer = Get-AzRecoveryServicesAsrProtectionContainer -Fabric $primaryFabric
$replicationPolicy = Get-AzRecoveryServicesAsrPolicy -Name "24-hour-policy"
$vm = Get-AzVM -ResourceGroupName "source-rg" -Name "sourceVM"
$protectableVM = Get-AzRecoveryServicesAsrProtectableItem -ProtectionContainer $protectionContainer -FriendlyName $vm.Name
New-AzRecoveryServicesAsrReplicationProtectedItem -ProtectableItem $protectableVM -Name $vm.Name -ProtectionContainerMapping $containerMapping -RecoveryAzureStorageAccountId $cacheStorageAccount.Id -RecoveryResourceGroupId $targetRG.ResourceId
```

```bash
# Azure CLI: Create Recovery Services Vault
rg=backup-rg
loc=eastus
vault=prodVault

az backup vault create --resource-group $rg --name $vault --location $loc --backup-storage-redundancy GeoRedundant

# Enable VM Backup
vm=myVM
vmrg=vm-rg
policy=DefaultPolicy

az backup protection enable-for-vm --resource-group $rg --vault-name $vault --vm $(az vm show -g $vmrg -n $vm --query id -o tsv) --policy-name $policy

# Trigger On-Demand Backup
container=$(az backup container list --resource-group $rg --vault-name $vault --backup-management-type AzureIaasVM --query "[0].name" -o tsv)
item=$(az backup item list --resource-group $rg --vault-name $vault --container-name $container --backup-management-type AzureIaasVM --workload-type VM --query "[0].name" -o tsv)
az backup protection backup-now --resource-group $rg --vault-name $vault --container-name $container --item-name $item --backup-management-type AzureIaasVM --workload-type VM --retain-until 01-01-2026

# Restore VM
rp=$(az backup recoverypoint list --resource-group $rg --vault-name $vault --container-name $container --item-name $item --backup-management-type AzureIaasVM --workload-type VM --query "[0].name" -o tsv)
az backup restore restore-disks --resource-group $rg --vault-name $vault --container-name $container --item-name $item --rp-name $rp --storage-account restoresa --target-resource-group restore-rg

# Enable Soft Delete
az backup vault backup-properties set --name $vault --resource-group $rg --soft-delete-feature-state Enable --soft-delete-retention-days 30
```

---

## Exam Tips

- **Recovery Services Vault vs Backup Vault:** RSV for VMs/SQL/Files/on-prem; Backup Vault for disks/blobs/PostgreSQL/AKS.
- **Backup Redundancy:** GRS default (cross-region DR); LRS cheaper (no cross-region restore).
- **Instant Restore:** Snapshots retained 1-5 days (faster restore); separate from vault retention.
- **Soft Delete:** 14-180 days; always-on prevents disable; doesn't count toward billing after 14 days.
- **Immutable Vaults:** Locked is irreversible; prevents deletion/modification; required for compliance.
- **MARS Agent:** Passphrase required for restore; store offline securely; cannot recover without it.
- **SQL Backup:** Full/Differential/Log backups; log every 15 min to 24 hrs; auto-protection for new DBs.
- **Azure Files:** Snapshot-based (no vault transfer); max 200 shares per storage account.
- **Site Recovery RPO:** 30 seconds (Hyper-V), continuous (Azure VM, VMware); RTO SLA ~1 hour.
- **Failover Types:** Test (non-disruptive), Planned (zero data loss), Unplanned (disaster, may lose recent data).
- **Cross-Region Restore:** Requires GRS/RA-GRS; restores from secondary (paired) region.

---

## Scenarios

### Scenario 1: Backup Azure VMs with 30-Day Retention

- **Requirement:** Daily backup of production VMs; 30-day retention; 2-day instant restore.
- **Solution:**
  1. Create **Recovery Services vault** with GRS redundancy.
  2. Use **Default policy** (daily backup, 30 days retention, 2 days instant restore).
  3. Enable backup on VMs via portal or PowerShell.
  4. Configure alerts for failed backups (Azure Monitor).

### Scenario 2: SQL Server Backup with Point-in-Time Restore

- **Requirement:** Backup SQL databases; restore to any point within last 7 days.
- **Solution:**
  1. Create **Recovery Services vault**.
  2. Discover SQL Server in Azure VM.
  3. Create policy: Full (daily), Log (every 15 min), 7-day retention.
  4. Enable backup for SQL databases.
  5. Restore: Select recovery point (specific date/time); logs provide point-in-time recovery.

### Scenario 3: On-Premises File Backup with MARS

- **Requirement:** Backup on-premises Windows Server files; 90-day retention.
- **Solution:**
  1. Create **Recovery Services vault** with GRS.
  2. Download and install **MARS agent** on Windows Server.
  3. Register server with vault credentials.
  4. Set encryption passphrase (store offline).
  5. Configure backup: Select files/folders, daily schedule, 90-day retention.
  6. Restore: Use MARS console, select recovery point, enter passphrase.

### Scenario 4: Disaster Recovery for Multi-Tier App

- **Requirement:** Replicate 3-tier app (SQL, App, Web) to secondary region; 1-hour RTO.
- **Solution:**
  1. Create **Recovery Services vault** in target region.
  2. Enable **Site Recovery** replication for all VMs (source → target).
  3. Create **Recovery plan** with VM groups: SQL (boot first), App, Web (boot last).
  4. Add **runbooks** for DNS updates, load balancer reconfiguration.
  5. Perform **Test failover** quarterly (isolated VNet).
  6. During disaster: **Unplanned failover** (latest app-consistent), commit, validate.
  7. After primary region recovers: **Reprotect** and **Failback**.

### Scenario 5: Protect Backups from Ransomware

- **Requirement:** Prevent malicious deletion of backups; regulatory compliance.
- **Solution:**
  1. Enable **Soft delete** with 90-day retention (always-on).
  2. Enable **Immutable vault** (locked) to prevent backup deletion.
  3. Use **CMK encryption** with Azure Key Vault for data sovereignty.
  4. Configure **RBAC**: Least privilege (Backup Contributor for admins, Backup Reader for auditors).
  5. Enable **MFA** for vault access (Azure AD conditional access).
  6. Monitor with **Azure Monitor alerts** (deletion attempts, failed backups).

---

## Troubleshooting

### Backup Job Fails (UserErrorVmNotFound)

- **Causes:** VM deleted, moved, or renamed after backup configuration.
- **Actions:** Disable backup for missing VM; re-enable if VM moved to different resource group.

### VM Extension Installation Fails

- **Causes:** VM agent not running, insufficient permissions, network restrictions.
- **Actions:** Verify VM agent status (`Get-AzVM -Status`); ensure VM has internet access (for Azure Backup endpoints); check NSG rules allow HTTPS to Azure Backup.

### MARS Agent Registration Fails

- **Causes:** Incorrect vault credentials, expired credentials, network issues.
- **Actions:** Re-download vault credentials (valid 48 hours); verify HTTPS connectivity to `*.backup.windowsazure.com`; check proxy settings.

### Restore Job Fails (Insufficient Permissions)

- **Causes:** Missing RBAC permissions on target resource group or storage account.
- **Actions:** Grant **Virtual Machine Contributor** on target resource group; assign **Storage Account Contributor** on staging storage account.

### Site Recovery Replication Lag High

- **Causes:** High VM disk churn, network bandwidth constraints, cache storage account throttling.
- **Actions:** Check cache storage account metrics (throttling); increase network bandwidth; reduce disk write rate on source VM; use Premium SSD for cache.

### SQL Backup Fails (InsufficientLogBackupRetention)

- **Causes:** SQL database recovery model changed from Full to Simple.
- **Actions:** Verify SQL database recovery model (must be Full or Bulk-logged for log backups); change back to Full; resume backup.

---

## Command Quick Reference

```powershell
# List all backup items in vault
$vault = Get-AzRecoveryServicesVault -Name "myVault" -ResourceGroupName "myRG"
Get-AzRecoveryServicesBackupItem -WorkloadType AzureVM -VaultId $vault.ID

# Get backup job status
Get-AzRecoveryServicesBackupJob -Status InProgress -VaultId $vault.ID

# Disable backup (retain data)
$item = Get-AzRecoveryServicesBackupItem -WorkloadType AzureVM -Name "myVM" -VaultId $vault.ID
Disable-AzRecoveryServicesBackupProtection -Item $item -VaultId $vault.ID

# Disable backup (delete data)
Disable-AzRecoveryServicesBackupProtection -Item $item -RemoveRecoveryPoints -Force -VaultId $vault.ID

# Site Recovery: Test Failover
$vm = Get-AzRecoveryServicesAsrReplicationProtectedItem -ProtectionContainer $protectionContainer -Name "myVM"
$recoveryPoint = Get-AzRecoveryServicesAsrRecoveryPoint -ReplicationProtectedItem $vm | Select-Object -First 1
$testNetwork = Get-AzVirtualNetwork -Name "test-vnet" -ResourceGroupName "test-rg"
Start-AzRecoveryServicesAsrTestFailoverJob -ReplicationProtectedItem $vm -RecoveryPoint $recoveryPoint -AzureVMNetworkId $testNetwork.Id

# Site Recovery: Commit Failover
$failoverJob = Start-AzRecoveryServicesAsrUnplannedFailoverJob -ReplicationProtectedItem $vm -RecoveryPoint $recoveryPoint -Direction PrimaryToRecovery
Wait-AzRecoveryServicesAsrJob -Job $failoverJob
$commitJob = Start-AzRecoveryServicesAsrCommitFailoverJob -ReplicationProtectedItem $vm
```

```bash
# List backup jobs
az backup job list --resource-group myRG --vault-name myVault --status InProgress

# Disable backup (retain data)
az backup protection disable --resource-group myRG --vault-name myVault --container-name <container> --item-name <item> --backup-management-type AzureIaasVM --workload-type VM --yes

# Disable backup (delete data)
az backup protection disable --resource-group myRG --vault-name myVault --container-name <container> --item-name <item> --backup-management-type AzureIaasVM --workload-type VM --delete-backup-data true --yes

# Get recovery points
az backup recoverypoint list --resource-group myRG --vault-name myVault --container-name <container> --item-name <item> --backup-management-type AzureIaasVM --workload-type VM
```

---

**Quick Recap**: Use **Recovery Services Vault** for Azure VM, SQL, Files, and on-premises backups; **Backup Vault** for disks/blobs/PostgreSQL/AKS; configure **soft delete** and **immutability** for protection against deletion; implement **Azure Site Recovery** for disaster recovery with replication, failover, and failback; follow backup best practices (GRS redundancy, test restores, secure passphrases) for business continuity.
