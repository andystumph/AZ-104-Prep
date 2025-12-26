# Configure Azure Files and Blob Storage

## Table of Contents
1. [Overview](#overview)
2. [Azure Files](#azure-files)
3. [Azure Blob Storage](#azure-blob-storage)
4. [Blob Access Tiers](#blob-access-tiers)
5. [Blob Lifecycle Management](#blob-lifecycle-management)
6. [Data Protection Features](#data-protection-features)
7. [PowerShell & CLI Commands](#powershell--cli-commands)
8. [Exam Tips](#exam-tips)
9. [Common Scenarios & Troubleshooting](#common-scenarios--troubleshooting)
10. [Related Topics](#related-topics)

---

## Overview

Azure Files and Blob Storage are key components of Azure Storage, each designed for specific use cases. This guide covers configuration, management, and optimization of both services for the AZ-104 exam.

### Service Comparison

| Feature | Azure Files | Azure Blob Storage |
|---------|-------------|-------------------|
| **Purpose** | File shares (SMB/NFS) | Object storage for unstructured data |
| **Protocols** | SMB 3.x, NFS 4.1, REST | REST, Data Lake Storage Gen2 |
| **Use Cases** | Lift-and-shift, shared storage, config files | Backups, media, logs, big data analytics |
| **Access Methods** | Mount as drive, REST API | REST API, SDKs, Storage Explorer |
| **Tiers** | Transaction optimized, Hot, Cool | Hot, Cool, Cold, Archive, Smart |
| **Max File/Blob** | 4 TiB | ~190.7 TiB (block blob) |
| **Authentication** | Storage key, Azure AD, AD DS | Storage key, SAS, Azure AD |

**Exam Weight:** Storage configuration is approximately 15-20% of the AZ-104 exam.

---

## Azure Files

Azure Files offers fully managed file shares in the cloud accessible via SMB and NFS protocols.

### File Share Types

#### Standard File Shares (HDD)
- **Performance tier:** Transaction optimized, Hot, or Cool
- **Storage type:** HDD-based (general-purpose v2 accounts)
- **Redundancy:** LRS, ZRS, GRS, GZRS
- **Max size:** 100 TiB (with large file shares enabled)
- **Billing:** Pay-as-you-go
- **Use case:** General-purpose file shares, cost-sensitive workloads

#### Premium File Shares (SSD)
- **Performance tier:** Premium only
- **Storage type:** SSD-based (FileStorage accounts)
- **Redundancy:** LRS, ZRS only
- **Max size:** 100 TiB
- **Billing:** Provisioned capacity (pay for provisioned size regardless of usage)
- **Use case:** High-performance, low-latency, I/O-intensive workloads

### Creating a File Share

#### Azure Portal

1. Navigate to **Storage Account**
2. Select **Data storage** → **File shares**
3. Click **+ File share**
4. Configure:
   - **Name:** Lowercase letters, numbers, hyphens (3-63 characters)
   - **Tier:** Transaction optimized, Hot, Cool (standard) or Premium
   - **Provisioned capacity:** Required for premium shares
   - **Backup:** Enable Azure Backup (optional)
5. Click **Create**

**Exam Tip:** Premium file shares require FileStorage account type, cannot be converted from standard.

### SMB File Shares

SMB (Server Message Block) is the most common protocol for Azure Files.

#### SMB Protocol Versions

| Version | Supported OS | Features |
|---------|-------------|----------|
| **SMB 3.1.1** | Windows 10+, Windows Server 2016+ | AES-256-GCM encryption, pre-auth integrity |
| **SMB 3.0** | Windows 8+, Windows Server 2012+ | Encryption, multi-channel |
| **SMB 2.1** | Windows 7, Windows Server 2008 R2 | Legacy (not recommended) |

**Default:** Encryption in transit enabled (SMB 3.x with encryption)

#### Authentication Methods

**1. Storage Account Key (Default)**
- Uses storage account access key
- Simple but less secure
- Grants full access to file share

**2. Azure AD Kerberos (Identity-based)**
- Integrates with Azure AD
- Supports Azure AD Domain Services or on-premises AD DS
- Enables NTFS permissions and ACLs
- **Recommended for production**

**3. On-premises AD DS**
- Hybrid identity integration
- Domain-joined machines authenticate with AD credentials
- Requires AD DS sync to Azure

#### Mounting SMB File Shares

**Windows:**

```powershell
# Using storage account key
$connectTestResult = Test-NetConnection -ComputerName <storage-account>.file.core.windows.net -Port 445
if ($connectTestResult.TcpTestSucceeded) {
    $username = "Azure\<storage-account>"
    $password = "<storage-account-key>"
    
    # Create credential
    $securePassword = ConvertTo-SecureString $password -AsPlainText -Force
    $credential = New-Object System.Management.Automation.PSCredential($username, $securePassword)
    
    # Mount the file share
    New-PSDrive -Name Z -PSProvider FileSystem -Root \\<storage-account>.file.core.windows.net\<share-name> -Credential $credential -Persist
} else {
    Write-Error "Port 445 blocked. Cannot connect to Azure Files."
}
```

**Persistent mount with cmdkey:**

```cmd
cmdkey /add:<storage-account>.file.core.windows.net /user:Azure\<storage-account> /pass:<storage-account-key>
net use Z: \\<storage-account>.file.core.windows.net\<share-name>
```

**Linux (SMB):**

```bash
# Install cifs-utils
sudo apt-get install cifs-utils  # Ubuntu/Debian
sudo yum install cifs-utils      # RHEL/CentOS

# Create mount point
sudo mkdir /mnt/azurefileshare

# Mount with credentials
sudo mount -t cifs //<storage-account>.file.core.windows.net/<share-name> /mnt/azurefileshare -o username=<storage-account>,password=<storage-account-key>,serverino,nosharesock,actimeo=30

# Persistent mount in /etc/fstab
echo "//<storage-account>.file.core.windows.net/<share-name> /mnt/azurefileshare cifs credentials=/etc/smbcredentials,serverino,nosharesock,actimeo=30" | sudo tee -a /etc/fstab

# Create credentials file
sudo bash -c 'echo "username=<storage-account>" >> /etc/smbcredentials'
sudo bash -c 'echo "password=<storage-account-key>" >> /etc/smbcredentials'
sudo chmod 600 /etc/smbcredentials
```

### NFS File Shares

NFS (Network File System) protocol for Linux workloads.

#### NFS Requirements

- **Account type:** FileStorage (premium) or general-purpose v2 (standard)
- **Protocol:** NFS 4.1 only
- **OS:** Linux kernel 4.3+
- **Redundancy:** LRS or ZRS only (no GRS/GZRS)
- **Network:** Private endpoint or service endpoint required
- **Authentication:** Host-based (no Azure AD integration)

#### Creating NFS File Share

**Portal:**
1. Storage account → **File shares** → **+ File share**
2. Name: `nfs-share`
3. **Protocol:** Select **NFS**
4. **Tier:** Premium (SSD)
5. Click **Create**

**Important:** Must disable "Secure transfer required" for NFS shares.

#### Mounting NFS File Shares

**Linux:**

```bash
# Install nfs-common
sudo apt-get install nfs-common  # Ubuntu/Debian
sudo yum install nfs-utils       # RHEL/CentOS

# Create mount point
sudo mkdir -p /mnt/nfsshare

# Mount NFS share
sudo mount -t nfs -o vers=4,minorversion=1,sec=sys <storage-account>.file.core.windows.net:/<storage-account>/<share-name> /mnt/nfsshare

# Persistent mount in /etc/fstab
echo "<storage-account>.file.core.windows.net:/<storage-account>/<share-name> /mnt/nfsshare nfs vers=4,minorversion=1,sec=sys 0 0" | sudo tee -a /etc/fstab
```

**Performance optimization with nconnect:**

```bash
sudo mount -t nfs -o vers=4,minorversion=1,sec=sys,nconnect=4 <storage-account>.file.core.windows.net:/<storage-account>/<share-name> /mnt/nfsshare
```

### File Share Tiers

| Tier | Use Case | Billing | IOPS | Throughput |
|------|----------|---------|------|------------|
| **Transaction Optimized** | High transaction rate | Storage + transaction costs | Up to 10,000 | Up to 300 MiB/s |
| **Hot** | General-purpose active data | Moderate storage, lower transaction | Up to 10,000 | Up to 300 MiB/s |
| **Cool** | Infrequently accessed data | Lower storage, higher transaction | Up to 10,000 | Up to 300 MiB/s |
| **Premium** | Low latency, high throughput | Provisioned capacity | Up to 100,000 | Up to 10 GiB/s |

**Changing tier:**
1. File share → **Configuration**
2. **Access tier:** Select new tier
3. Click **Save**

**Exam Scenario:** Shared storage for lift-and-shift application with 500 users accessing files. Answer: Azure Files with SMB protocol and transaction-optimized tier.

### File Share Snapshots

Point-in-time read-only copies of file shares.

#### Creating Snapshots

**Portal:**
1. File share → **Operations** → **Snapshots**
2. Click **+ Add snapshot**
3. Optional comment
4. Click **OK**

**Retention:** Manual deletion (no automatic expiration)

**Use Cases:**
- Protection against accidental deletion
- Protection against accidental modification
- Backup and recovery
- Testing and development

#### Restoring from Snapshots

**Restore entire share:**
1. File share → **Snapshots**
2. Select snapshot → **Restore**
3. Choose **Restore share** or **Overwrite share**
4. Click **OK**

**Restore individual files:**
1. File share → **Snapshots**
2. Select snapshot → **Browse Files**
3. Navigate to file → **Restore**
4. Choose destination

**Exam Tip:** Snapshots are incremental. Only changes since last snapshot consume additional storage.

### Azure File Sync

Extends on-premises file servers to Azure Files.

**Benefits:**
- Centralize file shares in Azure
- Cloud tiering (free up on-premises space)
- Multi-site access and sync
- Disaster recovery

**Components:**
- **Storage Sync Service:** Azure resource coordinating sync
- **Sync Group:** Defines topology (cloud endpoint + server endpoints)
- **Server Endpoint:** Path on registered Windows Server
- **Cloud Endpoint:** Azure file share

**Exam Scenario:** Central office and 10 branch offices need shared files with local caching. Answer: Azure File Sync with cloud tiering.

---

## Azure Blob Storage

Azure Blob Storage is optimized for storing massive amounts of unstructured data.

### Blob Types

#### Block Blobs
- **Purpose:** Text and binary data (documents, images, videos)
- **Max size:** ~190.7 TiB (50,000 blocks × 4000 MiB)
- **Upload method:** Blocks committed as single blob
- **Optimization:** Parallel uploads, efficient updates
- **Use cases:** Streaming, backup, logs, general-purpose storage

#### Append Blobs
- **Purpose:** Append operations only (logging, telemetry)
- **Max size:** ~195 GiB
- **Operations:** Append blocks only (no update/delete of existing blocks)
- **Use cases:** Log files, audit logs, streaming data

#### Page Blobs
- **Purpose:** Random read/write operations (VHD files)
- **Max size:** 8 TiB
- **Optimization:** Optimized for frequent random access
- **Use cases:** VM disks (OS and data disks), database files

### Creating Blob Containers

#### Azure Portal

1. Storage account → **Data storage** → **Containers**
2. Click **+ Container**
3. Configure:
   - **Name:** Lowercase, numbers, hyphens (3-63 characters)
   - **Public access level:**
     - **Private:** No anonymous access (default, recommended)
     - **Blob:** Anonymous read for blobs only
     - **Container:** Anonymous read for blobs and container metadata
4. Click **Create**

**Exam Tip:** Default is Private. Never use Blob or Container access for sensitive data.

### Uploading Blobs

#### Portal Upload

1. Container → **Upload**
2. Select files
3. Configure:
   - **Blob type:** Block blob (default)
   - **Block size:** 4 MB default (max 100 MB)
   - **Access tier:** Hot, Cool, Cold, Archive
   - **Upload to folder:** Optional virtual directory
4. Click **Upload**

#### PowerShell Upload

```powershell
# Upload single file
Set-AzStorageBlobContent `
  -Container 'mycontainer' `
  -File 'C:\files\document.pdf' `
  -Blob 'documents/document.pdf' `
  -Context $ctx `
  -StandardBlobTier 'Hot'

# Upload directory recursively
Get-ChildItem -Path 'C:\uploads' -File -Recurse | ForEach-Object {
    $blobName = $_.FullName.Substring('C:\uploads\'.Length).Replace('\', '/')
    Set-AzStorageBlobContent `
        -Container 'mycontainer' `
        -File $_.FullName `
        -Blob $blobName `
        -Context $ctx
}
```

### Blob Properties and Metadata

**System Properties (read-only):**
- `Content-Type`: MIME type (e.g., `application/pdf`, `image/jpeg`)
- `Content-Length`: Size in bytes
- `Last-Modified`: Last modification timestamp
- `ETag`: Concurrency control identifier

**Metadata (custom key-value pairs):**
- Up to 8 KB per blob
- Stored with blob, returned in GET operations
- Use cases: Classification, indexing, custom tags

**Setting metadata:**

```powershell
# PowerShell
$blob = Get-AzStorageBlob -Container 'mycontainer' -Blob 'file.txt' -Context $ctx
$blob.ICloudBlob.Metadata.Add('Department', 'Finance')
$blob.ICloudBlob.Metadata.Add('Project', 'Q4-2025')
$blob.ICloudBlob.SetMetadata()
```

---

## Blob Access Tiers

Access tiers optimize storage costs based on data access patterns.

### Tier Overview

#### Hot Tier
- **Access pattern:** Frequently accessed data
- **Storage cost:** Highest
- **Access cost:** Lowest
- **Latency:** Milliseconds (online)
- **Min retention:** None
- **Use cases:** Active data, frequent reads/writes, streaming media
- **Availability:** 99.9% (RA-GRS: 99.99%)

#### Cool Tier
- **Access pattern:** Infrequently accessed data
- **Storage cost:** Lower than Hot
- **Access cost:** Higher than Hot
- **Latency:** Milliseconds (online)
- **Min retention:** 30 days (early deletion penalty if deleted sooner)
- **Use cases:** Short-term backup, data for analysis, infrequent access
- **Availability:** 99% (RA-GRS: 99.9%)

#### Cold Tier
- **Access pattern:** Rarely accessed data
- **Storage cost:** Lower than Cool
- **Access cost:** Higher than Cool
- **Latency:** Milliseconds (online)
- **Min retention:** 90 days (early deletion penalty)
- **Use cases:** Long-term backup, compliance data, archival
- **Availability:** 99% (RA-GRS: 99.9%)

#### Archive Tier
- **Access pattern:** Rarely accessed with flexible latency
- **Storage cost:** Lowest (up to 10x cheaper than Hot)
- **Access cost:** Highest
- **Latency:** Hours (offline tier - must rehydrate)
- **Min retention:** 180 days (early deletion penalty)
- **Use cases:** Long-term archival, compliance retention, cold backups
- **Availability:** 99% (RA-GRS: 99.9%)
- **Redundancy:** LRS, GRS, RA-GRS only (not ZRS/GZRS)

**Exam Tip:** Archive tier is offline. Blob must be rehydrated to online tier (Hot/Cool/Cold) before reading.

### Setting Blob Tier

#### On Upload

**Portal:**
- During upload → **Advanced** → **Access tier** → Select tier

**PowerShell:**

```powershell
Set-AzStorageBlobContent `
  -Container 'mycontainer' `
  -File 'C:\archive\old-data.zip' `
  -Blob 'archives/old-data.zip' `
  -Context $ctx `
  -StandardBlobTier 'Archive'
```

#### Change Existing Blob Tier

**Portal:**
1. Select blob → **Change tier**
2. Select new tier
3. **Rehydration priority** (if from Archive): Standard or High
4. Click **Save**

**PowerShell:**

```powershell
# Hot to Archive
$blob = Get-AzStorageBlob -Container 'mycontainer' -Blob 'file.pdf' -Context $ctx
$blob.BlobClient.SetAccessTier('Archive', $null)

# Archive to Hot (rehydration)
$blob.BlobClient.SetAccessTier('Hot', $null, 'High')  # High priority
```

**Rehydration Priority:**
- **Standard:** Up to 15 hours
- **High:** Under 1 hour for objects <10 GB (higher cost)

### Tier Comparison Matrix

| Feature | Hot | Cool | Cold | Archive |
|---------|-----|------|------|---------|
| **Storage cost/GB** | $0.018 | $0.01 | $0.0045 | $0.00099 |
| **Access cost (read)** | Low | Medium | Higher | Highest |
| **Availability SLA** | 99.9% | 99% | 99% | 99% |
| **Min retention** | None | 30 days | 90 days | 180 days |
| **Latency** | <10 ms | <10 ms | <10 ms | Hours |
| **State** | Online | Online | Online | Offline |
| **Redundancy** | All | All | All | LRS/GRS only |

*Pricing is approximate and region-dependent. Check Azure pricing calculator for exact costs.*

**Exam Scenario:** 500 TB of data accessed once per year, must be retained for 7 years. Solution: Archive tier (lowest storage cost, acceptable latency).

### Default Account-Level Tier

Set default tier for new blobs (if tier not specified at upload):

1. Storage account → **Configuration**
2. **Blob access tier:** Hot or Cool (not Archive)
3. Click **Save**

**Note:** Does not change existing blobs, only affects new uploads without explicit tier.

---

## Blob Lifecycle Management

Automatically transition blobs between tiers or delete blobs based on rules.

### Lifecycle Policy Overview

**Purpose:**
- Optimize costs by automatically moving data to cooler tiers
- Delete expired or old data automatically
- Reduce manual management overhead

**Actions:**
- **tierToCool:** Move to Cool tier
- **tierToCold:** Move to Cold tier
- **tierToArchive:** Move to Archive tier
- **delete:** Delete blob
- **enableAutoTierToHotFromCool:** Automatically move Cool → Hot on access

**Conditions:**
- `daysAfterModificationGreaterThan`: Days since last modification
- `daysAfterCreationGreaterThan`: Days since creation (versions/snapshots)
- `daysAfterLastAccessTimeGreaterThan`: Days since last access (requires last access tracking)
- `daysAfterLastTierChangeGreaterThan`: Days since tier change

### Creating Lifecycle Policy

#### Azure Portal

1. Storage account → **Data management** → **Lifecycle management**
2. Click **Add rule**
3. **Details:**
   - **Rule name:** Descriptive name
   - **Rule scope:** All blobs or limit by filters
4. **Base blobs:**
   - Actions: Tier to Cool/Cold/Archive, Delete
   - Conditions: Days after modification/access
5. **Filters (optional):**
   - **Blob prefix:** e.g., `logs/`, `backups/`
   - **Blob types:** Block blob, Append blob
6. Click **Add**

#### PowerShell

```powershell
# Create action object
$action = Add-AzStorageAccountManagementPolicyAction `
  -BaseBlobAction Delete `
  -daysAfterModificationGreaterThan 180

# Add tier to cool
Add-AzStorageAccountManagementPolicyAction `
  -InputObject $action `
  -BaseBlobAction TierToCool `
  -daysAfterModificationGreaterThan 30

# Add tier to archive
Add-AzStorageAccountManagementPolicyAction `
  -InputObject $action `
  -BaseBlobAction TierToArchive `
  -daysAfterModificationGreaterThan 90

# Create filter
$filter = New-AzStorageAccountManagementPolicyFilter `
  -PrefixMatch 'logs/' `
  -BlobType blockBlob

# Create rule
$rule = New-AzStorageAccountManagementPolicyRule `
  -Name 'move-logs-to-archive' `
  -Action $action `
  -Filter $filter

# Apply policy
Set-AzStorageAccountManagementPolicy `
  -ResourceGroupName 'rg-storage' `
  -StorageAccountName 'storageaccount01' `
  -Rule $rule
```

### Lifecycle Policy Examples

#### Example 1: Age-based tiering and deletion

```json
{
  "rules": [
    {
      "name": "archive-old-data",
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["container1/"]
        },
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 30 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 90 },
            "delete": { "daysAfterModificationGreaterThan": 365 }
          }
        }
      }
    }
  ]
}
```

#### Example 2: Last access time-based with auto-tier to hot

```json
{
  "rules": [
    {
      "name": "optimize-by-access",
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": ["blockBlob"]
        },
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterLastAccessTimeGreaterThan": 30 },
            "enableAutoTierToHotFromCool": true
          }
        }
      }
    }
  ]
}
```

**Important:** `enableAutoTierToHotFromCool` requires last access time tracking enabled (incurs additional cost).

#### Example 3: Snapshot and version management

```json
{
  "rules": [
    {
      "name": "manage-versions",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "delete": { "daysAfterModificationGreaterThan": 365 }
          },
          "snapshot": {
            "delete": { "daysAfterCreationGreaterThan": 90 }
          },
          "version": {
            "tierToArchive": { "daysAfterCreationGreaterThan": 90 },
            "delete": { "daysAfterCreationGreaterThan": 365 }
          }
        }
      }
    }
  ]
}
```

**Exam Tip:** Lifecycle policies run once per day. Changes may take up to 24 hours to apply.

### Last Access Time Tracking

Enable to use `daysAfterLastAccessTimeGreaterThan` condition:

1. Storage account → **Data management** → **Lifecycle management**
2. Enable **Access tracking**
3. Click **Save**

**Cost:** Small additional charge for tracking access times.

**Use case:** Automatically tier infrequently accessed data to Cool, move back to Hot when accessed.

---

## Data Protection Features

### Soft Delete for Blobs

Protects blobs from accidental deletion and modification.

#### Enabling Soft Delete

**Portal:**
1. Storage account → **Data protection**
2. **Recovery** section → Enable **Enable soft delete for blobs**
3. **Retention days:** 1-365 (recommended: 7-30 days)
4. Click **Save**

**PowerShell:**

```powershell
Enable-AzStorageBlobDeleteRetentionPolicy `
  -ResourceGroupName 'rg-storage' `
  -StorageAccountName 'storageaccount01' `
  -RetentionDays 7
```

#### Restoring Soft-Deleted Blobs

**Portal:**
1. Container → Enable **Show deleted blobs**
2. Select deleted blob
3. Click **Undelete**

**PowerShell:**

```powershell
# List deleted blobs
Get-AzStorageBlob -Container 'mycontainer' -Context $ctx -IncludeDeleted

# Restore deleted blob
$blob = Get-AzStorageBlob -Container 'mycontainer' -Blob 'file.txt' -Context $ctx -IncludeDeleted
$blob.ICloudBlob.Undelete()
```

**Behavior:**
- Deleted blobs retained for specified retention period
- After retention expires, permanently deleted
- Soft-deleted blobs count toward storage quota

### Soft Delete for Containers

Protects entire containers from deletion.

**Enabling:**
1. Storage account → **Data protection**
2. Enable **Enable soft delete for containers**
3. **Retention days:** 1-365
4. Click **Save**

**Restoring:**
1. Storage account → **Containers** → **Show deleted containers**
2. Select deleted container → **Undelete**

**Exam Tip:** Soft delete for containers and blobs work independently. Enable both for comprehensive protection.

### Blob Versioning

Automatically maintains previous versions of blobs.

#### Enabling Versioning

**Portal:**
1. Storage account → **Data protection**
2. **Tracking** section → Enable **Enable versioning for blobs**
3. Click **Save**

**PowerShell:**

```powershell
Update-AzStorageBlobServiceProperty `
  -ResourceGroupName 'rg-storage' `
  -StorageAccountName 'storageaccount01' `
  -IsVersioningEnabled $true
```

#### How Versioning Works

**On blob modification:**
- Current version → becomes previous version (with version ID)
- New content → becomes current version

**On blob deletion:**
- Current version → becomes previous version
- Delete marker placed (no current version exists)

**Restoring version:**
```powershell
# List versions
Get-AzStorageBlob -Container 'mycontainer' -Blob 'file.txt' -Context $ctx -IncludeVersion

# Promote version to current
Copy-AzStorageBlob `
  -SrcContainer 'mycontainer' `
  -SrcBlob 'file.txt' `
  -DestContainer 'mycontainer' `
  -DestBlob 'file.txt' `
  -Context $ctx `
  -SrcVersionId '<version-id>'
```

**Cost Considerations:**
- Each version consumes storage (like separate blob)
- Use lifecycle policy to delete old versions
- Recommended: Combine with soft delete for comprehensive protection

### Point-in-Time Restore

Restore all block blobs in account/container to specific point in time.

#### Requirements

Must enable (in order):
1. Blob soft delete (retention ≥ restore period + 1 day)
2. Change feed
3. Blob versioning
4. Point-in-time restore

**Enabling:**

**Portal:**
1. Storage account → **Data protection**
2. Enable **Turn on point-in-time restore**
3. **Maximum restore point:** 1-365 days (must be < soft delete retention)
4. Click **Save**

**PowerShell:**

```powershell
# Enable prerequisites
Enable-AzStorageBlobDeleteRetentionPolicy -ResourceGroupName 'rg-storage' -StorageAccountName 'storageaccount01' -RetentionDays 14
Update-AzStorageBlobServiceProperty -ResourceGroupName 'rg-storage' -StorageAccountName 'storageaccount01' -EnableChangeFeed $true
Update-AzStorageBlobServiceProperty -ResourceGroupName 'rg-storage' -StorageAccountName 'storageaccount01' -IsVersioningEnabled $true

# Enable point-in-time restore
Enable-AzStorageBlobRestorePolicy -ResourceGroupName 'rg-storage' -StorageAccountName 'storageaccount01' -RestoreDays 7
```

#### Performing Restore

**Portal:**
1. Storage account → **Data protection**
2. **Point-in-time restore** section → **Restore containers**
3. Select restore point (date/time in UTC)
4. Choose **Restore all containers** or specific ranges
5. Click **Restore**

**Limitations:**
- Block blobs only (not append/page blobs)
- Cannot restore archived blobs
- Container deletion cannot be restored (use container soft delete)
- Restore time depends on data volume (hours for large datasets)

**Exam Scenario:** Accidentally deleted/modified thousands of blobs yesterday. Solution: Point-in-time restore to 48 hours ago.

### Immutable Storage (WORM)

Write-Once-Read-Many (WORM) storage for compliance and legal hold.

**Types:**
1. **Time-based retention:** Blobs cannot be deleted/modified for specified period
2. **Legal hold:** Blobs protected indefinitely until hold removed

**Use cases:**
- Financial records (SOX, FINRA)
- Healthcare records (HIPAA)
- Legal documents (e-discovery)

**Scope:** Container-level or version-level policies

**Exam Tip:** Immutable blobs cannot be deleted or modified even by account owner. Plan carefully before enabling.

---

## PowerShell & CLI Commands

### Azure Files Commands

#### PowerShell

```powershell
# Create file share
New-AzRmStorageShare `
  -ResourceGroupName 'rg-storage' `
  -StorageAccountName 'storageaccount01' `
  -Name 'myfileshare' `
  -AccessTier 'Hot' `
  -QuotaGiB 100

# Get file share
Get-AzRmStorageShare `
  -ResourceGroupName 'rg-storage' `
  -StorageAccountName 'storageaccount01' `
  -Name 'myfileshare'

# Update file share tier
Update-AzRmStorageShare `
  -ResourceGroupName 'rg-storage' `
  -StorageAccountName 'storageaccount01' `
  -Name 'myfileshare' `
  -AccessTier 'Cool'

# Upload file
$ctx = (Get-AzStorageAccount -ResourceGroupName 'rg-storage' -Name 'storageaccount01').Context
Set-AzStorageFileContent `
  -ShareName 'myfileshare' `
  -Source 'C:\files\document.pdf' `
  -Path 'documents/document.pdf' `
  -Context $ctx

# Download file
Get-AzStorageFileContent `
  -ShareName 'myfileshare' `
  -Path 'documents/document.pdf' `
  -Destination 'C:\downloads\document.pdf' `
  -Context $ctx

# Create snapshot
$share = Get-AzRmStorageShare -ResourceGroupName 'rg-storage' -StorageAccountName 'storageaccount01' -Name 'myfileshare'
$snapshot = $share.Snapshot()

# List snapshots
Get-AzRmStorageShare `
  -ResourceGroupName 'rg-storage' `
  -StorageAccountName 'storageaccount01' `
  -Name 'myfileshare' `
  -IncludeSnapshot

# Delete file share
Remove-AzRmStorageShare `
  -ResourceGroupName 'rg-storage' `
  -StorageAccountName 'storageaccount01' `
  -Name 'myfileshare'
```

#### Azure CLI

```bash
# Create file share
az storage share create \
  --account-name storageaccount01 \
  --name myfileshare \
  --access-tier Hot \
  --quota 100

# Get file share
az storage share show \
  --account-name storageaccount01 \
  --name myfileshare

# Update file share tier
az storage share update \
  --account-name storageaccount01 \
  --name myfileshare \
  --access-tier Cool

# Upload file
az storage file upload \
  --account-name storageaccount01 \
  --share-name myfileshare \
  --source C:\files\document.pdf \
  --path documents/document.pdf

# Download file
az storage file download \
  --account-name storageaccount01 \
  --share-name myfileshare \
  --path documents/document.pdf \
  --dest C:\downloads\document.pdf

# Create snapshot
az storage share snapshot \
  --account-name storageaccount01 \
  --name myfileshare

# List snapshots
az storage share list \
  --account-name storageaccount01 \
  --include-snapshot

# Delete file share
az storage share delete \
  --account-name storageaccount01 \
  --name myfileshare
```

### Blob Storage Commands

#### PowerShell

```powershell
# Create container
New-AzStorageContainer `
  -Name 'mycontainer' `
  -Context $ctx `
  -Permission Off

# Upload blob with tier
Set-AzStorageBlobContent `
  -Container 'mycontainer' `
  -File 'C:\files\document.pdf' `
  -Blob 'documents/document.pdf' `
  -Context $ctx `
  -StandardBlobTier 'Cool' `
  -Properties @{'ContentType'='application/pdf'}

# Download blob
Get-AzStorageBlobContent `
  -Container 'mycontainer' `
  -Blob 'documents/document.pdf' `
  -Destination 'C:\downloads\document.pdf' `
  -Context $ctx

# List blobs
Get-AzStorageBlob -Container 'mycontainer' -Context $ctx

# Change blob tier
$blob = Get-AzStorageBlob -Container 'mycontainer' -Blob 'file.pdf' -Context $ctx
$blob.BlobClient.SetAccessTier('Archive', $null)

# Copy blob
Start-AzStorageBlobCopy `
  -SrcContainer 'source-container' `
  -SrcBlob 'file.txt' `
  -DestContainer 'dest-container' `
  -DestBlob 'file.txt' `
  -Context $ctx

# Delete blob
Remove-AzStorageBlob `
  -Container 'mycontainer' `
  -Blob 'document.pdf' `
  -Context $ctx

# Enable blob soft delete
Enable-AzStorageBlobDeleteRetentionPolicy `
  -ResourceGroupName 'rg-storage' `
  -StorageAccountName 'storageaccount01' `
  -RetentionDays 7

# Enable versioning
Update-AzStorageBlobServiceProperty `
  -ResourceGroupName 'rg-storage' `
  -StorageAccountName 'storageaccount01' `
  -IsVersioningEnabled $true

# List deleted blobs
Get-AzStorageBlob -Container 'mycontainer' -Context $ctx -IncludeDeleted

# Restore deleted blob
$blob = Get-AzStorageBlob -Container 'mycontainer' -Blob 'file.txt' -Context $ctx -IncludeDeleted
$blob.ICloudBlob.Undelete()
```

#### Azure CLI

```bash
# Create container
az storage container create \
  --account-name storageaccount01 \
  --name mycontainer \
  --public-access off

# Upload blob with tier
az storage blob upload \
  --account-name storageaccount01 \
  --container-name mycontainer \
  --name documents/document.pdf \
  --file C:\files\document.pdf \
  --tier Cool \
  --content-type application/pdf

# Download blob
az storage blob download \
  --account-name storageaccount01 \
  --container-name mycontainer \
  --name documents/document.pdf \
  --file C:\downloads\document.pdf

# List blobs
az storage blob list \
  --account-name storageaccount01 \
  --container-name mycontainer \
  --output table

# Change blob tier
az storage blob set-tier \
  --account-name storageaccount01 \
  --container-name mycontainer \
  --name file.pdf \
  --tier Archive

# Copy blob
az storage blob copy start \
  --account-name storageaccount01 \
  --destination-container dest-container \
  --destination-blob file.txt \
  --source-container source-container \
  --source-blob file.txt

# Delete blob
az storage blob delete \
  --account-name storageaccount01 \
  --container-name mycontainer \
  --name document.pdf

# Enable blob soft delete
az storage account blob-service-properties update \
  --account-name storageaccount01 \
  --resource-group rg-storage \
  --enable-delete-retention true \
  --delete-retention-days 7

# Enable versioning
az storage account blob-service-properties update \
  --account-name storageaccount01 \
  --resource-group rg-storage \
  --enable-versioning true

# List deleted blobs
az storage blob list \
  --account-name storageaccount01 \
  --container-name mycontainer \
  --include d \
  --output table

# Restore deleted blob
az storage blob undelete \
  --account-name storageaccount01 \
  --container-name mycontainer \
  --name file.txt
```

### Lifecycle Management Commands

#### PowerShell

```powershell
# Create lifecycle policy
$action = Add-AzStorageAccountManagementPolicyAction `
  -BaseBlobAction Delete `
  -daysAfterModificationGreaterThan 180

Add-AzStorageAccountManagementPolicyAction `
  -InputObject $action `
  -BaseBlobAction TierToCool `
  -daysAfterModificationGreaterThan 30

Add-AzStorageAccountManagementPolicyAction `
  -InputObject $action `
  -BaseBlobAction TierToArchive `
  -daysAfterModificationGreaterThan 90

$filter = New-AzStorageAccountManagementPolicyFilter `
  -PrefixMatch 'container1/' `
  -BlobType blockBlob

$rule = New-AzStorageAccountManagementPolicyRule `
  -Name 'lifecycle-rule' `
  -Action $action `
  -Filter $filter

Set-AzStorageAccountManagementPolicy `
  -ResourceGroupName 'rg-storage' `
  -StorageAccountName 'storageaccount01' `
  -Rule $rule

# Get lifecycle policy
Get-AzStorageAccountManagementPolicy `
  -ResourceGroupName 'rg-storage' `
  -StorageAccountName 'storageaccount01'

# Remove lifecycle policy
Remove-AzStorageAccountManagementPolicy `
  -ResourceGroupName 'rg-storage' `
  -StorageAccountName 'storageaccount01'
```

#### Azure CLI

```bash
# Create lifecycle policy from JSON file
az storage account management-policy create \
  --account-name storageaccount01 \
  --resource-group rg-storage \
  --policy @policy.json

# Get lifecycle policy
az storage account management-policy show \
  --account-name storageaccount01 \
  --resource-group rg-storage

# Delete lifecycle policy
az storage account management-policy delete \
  --account-name storageaccount01 \
  --resource-group rg-storage
```

---

## Exam Tips

### Key Concepts for AZ-104

1. **Azure Files Protocols:**
   - **SMB:** Windows/Linux, identity-based auth, encrypted by default
   - **NFS:** Linux only, premium tier, host-based auth, no encryption by default
   - **Exam Tip:** Cannot use both SMB and NFS on same file share

2. **File Share Tiers:**
   - **Standard:** Transaction optimized, Hot, Cool (HDD, pay-as-you-go)
   - **Premium:** SSD, low latency, provisioned capacity
   - **Exam Scenario:** High IOPS requirement → Premium file share

3. **Blob Types:**
   - **Block blob:** General-purpose (documents, images, videos)
   - **Append blob:** Append-only (logs, audit trails)
   - **Page blob:** Random access (VM disks, databases)
   - **Exam Tip:** Page blobs optimized for VHD files

4. **Access Tiers:**
   - **Hot:** Frequent access, highest storage cost, lowest access cost
   - **Cool:** 30-day min retention, lower storage cost
   - **Cold:** 90-day min retention, even lower storage cost
   - **Archive:** 180-day min retention, offline (hours to rehydrate)
   - **Exam Trap:** Archive tier is offline. Must rehydrate before reading.

5. **Lifecycle Management:**
   - Automatically moves blobs between tiers based on age/access
   - Runs once per day (24-hour delay possible)
   - Supports filters (prefix, blob type)
   - Can delete old data automatically
   - **Exam Scenario:** Move logs to archive after 90 days → Lifecycle policy

6. **Data Protection:**
   - **Soft delete:** Retention period 1-365 days, recoverable
   - **Versioning:** Automatic previous versions, storage cost per version
   - **Point-in-time restore:** Restore all blobs to specific time, requires soft delete + change feed + versioning
   - **Exam Tip:** Combine soft delete + versioning for comprehensive protection

7. **Mounting File Shares:**
   - **Windows:** Port 445 must be open, use cmdkey for persistent
   - **Linux SMB:** cifs-utils package, /etc/fstab for persistent
   - **Linux NFS:** nfs-common/nfs-utils, vers=4,minorversion=1
   - **Exam Scenario:** Cannot mount file share → Check port 445 firewall

8. **Rehydration:**
   - **Standard priority:** Up to 15 hours
   - **High priority:** <1 hour for <10 GB (higher cost)
   - **Method 1:** Change tier (Set Blob Tier)
   - **Method 2:** Copy to new blob with online tier
   - **Exam Tip:** Copying avoids early deletion penalty

### Common Exam Scenarios

**Scenario 1: Choose File Share Protocol**
- **Question:** Linux application needs shared storage with POSIX permissions
- **Answer:** NFS file share with premium tier
- **Why:** NFS supports POSIX, SMB uses Windows ACLs

**Scenario 2: Optimize Blob Storage Costs**
- **Question:** 100 TB of backups, accessed once per month, retained 1 year
- **Answer:** Cool tier with lifecycle policy
- **Why:** Cool tier for infrequent access, lifecycle to delete after 365 days

**Scenario 3: Long-term Archival**
- **Question:** Compliance requires 10-year retention, accessed rarely
- **Answer:** Archive tier
- **Why:** Lowest cost, offline acceptable for rare access, meets 180-day min retention

**Scenario 4: Protect Against Deletion**
- **Question:** Prevent accidental blob deletion, enable recovery for 30 days
- **Answer:** Enable blob soft delete with 30-day retention
- **Why:** Soft delete maintains deleted blobs for recovery

**Scenario 5: Lift-and-Shift File Server**
- **Question:** Migrate on-premises file server to Azure, support 1000 concurrent users
- **Answer:** Azure Files with SMB protocol and transaction-optimized tier
- **Why:** SMB compatible with Windows file sharing, transaction-optimized for high access

**Scenario 6: Rehydrate Archived Blob**
- **Question:** Need to access blob in archive tier within 30 minutes
- **Answer:** Rehydrate with high priority to hot/cool tier
- **Why:** High priority provides <1 hour rehydration

**Scenario 7: Automatic Tiering**
- **Question:** Move blobs to cool tier after 30 days of inactivity, delete after 365 days
- **Answer:** Lifecycle management policy with daysAfterModificationGreaterThan
- **Why:** Automates tiering and deletion based on age

**Scenario 8: Restore Deleted Blobs**
- **Question:** Thousands of blobs accidentally deleted yesterday
- **Answer:** Point-in-time restore to 48 hours ago
- **Why:** Restores all blobs in container/account to specific point in time

---

## Common Scenarios & Troubleshooting

### Scenario: Cannot Mount Azure File Share

**Symptoms:**
- `mount error(115): Operation now in progress` (Linux)
- `System error 53` or `System error 1231` (Windows)
- Connection timeout

**Troubleshooting Steps:**
1. **Check port 445:** Many ISPs and organizations block SMB port 445
   ```powershell
   Test-NetConnection -ComputerName storageaccount.file.core.windows.net -Port 445
   ```
2. **Verify storage account accessible:** Check firewall rules
3. **Check credentials:** Verify storage account key is correct
4. **Verify secure transfer:** If NFS, disable "Secure transfer required"
5. **Check DNS resolution:** Ensure storage account FQDN resolves

**Resolution:**

**For port 445 blocked:**
- Use VPN or ExpressRoute to bypass ISP block
- Use Azure File Sync to sync to on-premises server
- Use SMB over QUIC (Windows Server 2022+ feature)

**For firewall restrictions:**
1. Storage account → **Networking** → **Firewalls and virtual networks**
2. Add client IP or use virtual network service endpoint
3. Click **Save**

### Scenario: Blob Lifecycle Policy Not Working

**Symptoms:**
- Blobs not moving to expected tier
- Blobs not deleted after expected time
- Policy shows in portal but no effect

**Troubleshooting Steps:**
1. **Check policy enabled:** Ensure policy not disabled
2. **Verify filters:** Prefix/blob type filters may exclude blobs
3. **Check timing:** Policies run once per day (wait 24-48 hours)
4. **Verify blob eligibility:** Append/page blobs don't support cool/archive tiers
5. **Check last modified date:** Policy uses UTC time, verify blob modification date
6. **Review policy JSON:** Syntax errors can prevent execution

**Resolution:**

```powershell
# Verify policy
Get-AzStorageAccountManagementPolicy -ResourceGroupName 'rg-storage' -StorageAccountName 'storageaccount01'

# Check blob last modified date
$blob = Get-AzStorageBlob -Container 'mycontainer' -Blob 'file.txt' -Context $ctx
$blob.LastModified

# Manually trigger tier change (policy will honor later)
$blob.BlobClient.SetAccessTier('Cool', $null)
```

### Scenario: Rehydration Takes Too Long

**Symptoms:**
- Archive blob rehydration exceeds expected time
- Need faster access to archived data

**Troubleshooting Steps:**
1. **Check priority:** Standard priority can take up to 15 hours
2. **Check blob size:** Large blobs (>10 GB) take longer even with high priority
3. **Verify rehydration status:**
   ```powershell
   $blob = Get-AzStorageBlob -Container 'mycontainer' -Blob 'file.zip' -Context $ctx
   $blob.BlobProperties.ArchiveStatus
   $blob.BlobProperties.RehydratePriority
   ```
4. **Check region:** Some regions may have longer rehydration times

**Resolution:**

**For faster access:**
1. Use high priority rehydration:
   ```powershell
   $blob.BlobClient.SetAccessTier('Hot', $null, 'High')
   ```
2. **Future prevention:** Don't archive frequently-accessed data
3. Consider Cool or Cold tier instead of Archive for better access time

### Scenario: Soft Delete Not Restoring Blobs

**Symptoms:**
- Deleted blobs not visible in "Show deleted blobs"
- Cannot undelete blob
- Deleted blob not in container

**Troubleshooting Steps:**
1. **Check soft delete enabled:** Verify enabled before deletion
2. **Check retention period:** Blob may have been deleted before soft delete enabled
3. **Check retention expired:** Retention period may have elapsed
4. **Verify view settings:** Portal filter may hide deleted blobs
5. **Check container:** Deleted container cannot be restored with blob soft delete (use container soft delete)

**Resolution:**

```powershell
# Verify soft delete configuration
Get-AzStorageBlobServiceProperty -ResourceGroupName 'rg-storage' -StorageAccountName 'storageaccount01'

# Enable "Show deleted blobs" in portal
# Or use PowerShell to list
Get-AzStorageBlob -Container 'mycontainer' -Context $ctx -IncludeDeleted | Where-Object {$_.IsDeleted -eq $true}

# Restore if found
$blob = Get-AzStorageBlob -Container 'mycontainer' -Blob 'file.txt' -Context $ctx -IncludeDeleted
$blob.ICloudBlob.Undelete()
```

**Prevention:**
- Enable both blob and container soft delete
- Use appropriate retention period (30+ days for important data)
- Enable versioning for additional protection

### Scenario: High Storage Costs with Versioning

**Symptoms:**
- Unexpected storage costs after enabling versioning
- Storage capacity growing rapidly
- Many versions accumulating

**Troubleshooting Steps:**
1. **Check version count:** Each version consumes storage
   ```powershell
   Get-AzStorageBlob -Container 'mycontainer' -Context $ctx -IncludeVersion | 
     Group-Object Name | Select-Object Name, Count
   ```
2. **Review blob modifications:** Frequent updates create many versions
3. **Check lifecycle policy:** May not be deleting old versions

**Resolution:**

**Implement lifecycle policy to delete old versions:**

```powershell
$action = Add-AzStorageAccountManagementPolicyAction -BlobVersionAction Delete -daysAfterCreationGreaterThan 90

$rule = New-AzStorageAccountManagementPolicyRule -Name 'delete-old-versions' -Action $action

Set-AzStorageAccountManagementPolicy -ResourceGroupName 'rg-storage' -StorageAccountName 'storageaccount01' -Rule $rule
```

**Best practices:**
- Set lifecycle policy to delete versions after 30-90 days
- Only enable versioning for critical data
- Consider using snapshots instead for backups (manual deletion control)

### Scenario: NFS Mount Permission Denied

**Symptoms:**
- `Permission denied` when accessing mounted NFS share
- Cannot write to NFS mount
- UID/GID mismatch errors

**Troubleshooting Steps:**
1. **Check mount options:** Verify `sec=sys` option used
2. **Check file ownership:** NFS uses UID/GID, may not match local users
3. **Verify network access:** NFS requires private endpoint or service endpoint
4. **Check no_root_squash:** Root user may be mapped to anonymous

**Resolution:**

```bash
# Remount with correct options
sudo umount /mnt/nfsshare
sudo mount -t nfs -o vers=4,minorversion=1,sec=sys <storage-account>.file.core.windows.net:/<storage-account>/<share-name> /mnt/nfsshare

# Check ownership
ls -la /mnt/nfsshare

# Change ownership if needed (may require root)
sudo chown -R user:group /mnt/nfsshare
```

**For persistent access:**
- Configure UID/GID mapping
- Use consistent user accounts across systems
- Consider Azure AD DS integration for SMB instead if identity management needed

---

## Related Topics

### Linked Study Materials

- **[Manage Storage](./Manage%20Storage.md):** Storage redundancy, AzCopy, object replication
- **[Secure Storage](./Secure%20Storage.md):** Security, encryption, network access control
- **[Implement Backup and Recovery](../Monitor%20and%20Backup%20Azure%20Resources/Implement%20Backup%20and%20Recovery.md):** Azure Backup for file shares
- **[Configure VMs](../Deploy%20and%20Manage%20Azure%20Compute%20Resources/Configure%20VMs.md):** VM disks and page blobs

### Azure Services Integration

- **Azure Backup:** Backup Azure Files shares with recovery vault
- **Azure File Sync:** Extend on-premises file servers to Azure Files
- **Azure CDN:** Cache blob content at edge locations for low latency
- **Azure Data Lake Storage Gen2:** Hierarchical namespace for big data
- **Azure Blob Storage + Azure Functions:** Event-driven processing on blob changes

### Advanced Topics (Beyond AZ-104)

- **Azure Files Identity-Based Authentication:** Azure AD DS, AD DS integration
- **Azure Blob Storage Premium:** High-performance block blob storage
- **Azure Data Lake Storage Gen2:** Hierarchical namespace, ACLs, analytics
- **Blob Index Tags:** Categorization and querying with tags
- **Storage Reserved Capacity:** Cost savings with 1-3 year commitments
- **Blob Inventory:** Generate reports of blob properties and metadata

### Documentation Links

- [Azure Files Documentation](https://learn.microsoft.com/en-us/azure/storage/files/)
- [Azure Blob Storage Documentation](https://learn.microsoft.com/en-us/azure/storage/blobs/)
- [Access Tiers Overview](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview)
- [Lifecycle Management](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview)
- [Soft Delete for Blobs](https://learn.microsoft.com/en-us/azure/storage/blobs/soft-delete-blob-overview)
- [Blob Versioning](https://learn.microsoft.com/en-us/azure/storage/blobs/versioning-overview)
- [Point-in-Time Restore](https://learn.microsoft.com/en-us/azure/storage/blobs/point-in-time-restore-overview)

---

**Study Checkpoint:** You should now understand Azure Files configuration (SMB/NFS protocols, mounting, tiers), Blob Storage (types, containers, tiers), access tier optimization (Hot/Cool/Cold/Archive), lifecycle management policies, and data protection features (soft delete, versioning, point-in-time restore). Practice creating file shares, uploading blobs to different tiers, and configuring lifecycle policies.

**Next Steps:** Review [Manage Storage](./Manage%20Storage.md) for redundancy and replication, and [Secure Storage](./Secure%20Storage.md) for security configurations. Complete Storage domain, then move to Compute resources.
