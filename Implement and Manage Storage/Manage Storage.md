# Manage Storage

## Table of Contents
1. [Overview](#overview)
2. [Storage Redundancy](#storage-redundancy)
3. [Storage Account Management](#storage-account-management)
4. [AzCopy](#azcopy)
5. [Azure Storage Explorer](#azure-storage-explorer)
6. [Object Replication](#object-replication)
7. [Import/Export Service](#importexport-service)
8. [Storage Account Failover](#storage-account-failover)
9. [Performance and Scalability](#performance-and-scalability)
10. [PowerShell & CLI Commands](#powershell--cli-commands)
11. [Exam Tips](#exam-tips)
12. [Common Scenarios & Troubleshooting](#common-scenarios--troubleshooting)
13. [Related Topics](#related-topics)

---

## Overview

Azure Storage management encompasses redundancy configuration, data transfer tools, replication strategies, and operational tasks for maintaining storage accounts. This guide covers the essential management capabilities tested in the AZ-104 exam.

### Key Management Areas

| Area | Tools/Features | Purpose |
|------|---------------|---------|
| **Redundancy** | LRS, ZRS, GRS, GZRS, RA-GRS, RA-GZRS | Data durability and availability |
| **Data Transfer** | AzCopy, Storage Explorer, Import/Export | Move data to/from Azure Storage |
| **Replication** | Object replication, Geo-replication | Copy data between accounts or regions |
| **Failover** | Account failover, failback | Disaster recovery operations |
| **Monitoring** | Metrics, logs, alerts | Track performance and issues |
| **Optimization** | Performance tiers, access tiers | Cost and performance tuning |

**Exam Weight:** Storage management is approximately 15-20% of the AZ-104 exam.

---

## Storage Redundancy

Azure Storage always replicates data to ensure durability and high availability. Different redundancy options provide varying levels of protection.

### Redundancy Options

#### Locally Redundant Storage (LRS)

**Characteristics:**
- **Copies:** 3 synchronous copies within single datacenter
- **Durability:** 99.999999999% (11 nines) per year
- **Availability:** 99.9% for read/write operations
- **Cost:** Lowest cost option
- **Protection:** Server rack and drive failures
- **Risk:** Entire datacenter disaster (fire, flood)

**Use Cases:**
- Non-critical data that can be reconstructed
- Data with frequent changes (temporary working data)
- Compliance restrictions prevent geo-replication
- Cost-sensitive scenarios

**Portal Configuration:**
1. Storage account creation → **Redundancy** → Select **Locally-redundant storage (LRS)**

#### Zone-Redundant Storage (ZRS)

**Characteristics:**
- **Copies:** 3 synchronous copies across 3 availability zones
- **Durability:** 99.9999999999% (12 nines) per year
- **Availability:** 99.9% for read/write operations
- **Cost:** Moderate (more than LRS, less than GRS)
- **Protection:** Datacenter failures, zone-level outages
- **Risk:** Region-wide disaster

**Use Cases:**
- High availability applications
- Production workloads
- Applications requiring consistency across zones
- No downtime for zone failures

**Portal Configuration:**
1. Storage account creation → **Redundancy** → Select **Zone-redundant storage (ZRS)**

**Exam Tip:** ZRS protects against datacenter failure but keeps data in same region. For regional disaster recovery, use GRS or GZRS.

#### Geo-Redundant Storage (GRS)

**Characteristics:**
- **Primary:** 3 copies with LRS in primary region
- **Secondary:** 3 copies with LRS in paired secondary region (hundreds of miles away)
- **Total copies:** 6 copies across 2 regions
- **Durability:** 99.99999999999999% (16 nines) per year
- **Availability:** 99.9% read/write (primary), 99.9% read (with RA-GRS)
- **RPO:** ~15 minutes (asynchronous replication)
- **Cost:** Higher cost

**Use Cases:**
- Mission-critical applications
- Disaster recovery requirements
- Regional compliance needs
- High durability requirements

**Portal Configuration:**
1. Storage account creation → **Redundancy** → Select **Geo-redundant storage (GRS)**

**Important:** Secondary region data is read-only by default. Enable **Read-access geo-redundant storage (RA-GRS)** for read access to secondary.

#### Geo-Zone-Redundant Storage (GZRS)

**Characteristics:**
- **Primary:** 3 copies across 3 availability zones (ZRS)
- **Secondary:** 3 copies with LRS in paired secondary region
- **Total copies:** 6 copies across 2 regions and multiple zones
- **Durability:** 99.99999999999999% (16 nines) per year
- **Availability:** 99.99% read/write (primary)
- **RPO:** ~15 minutes
- **Cost:** Highest cost

**Use Cases:**
- Maximum availability and durability
- Mission-critical applications
- Zone and region failure protection
- Enterprise applications

**Portal Configuration:**
1. Storage account creation → **Redundancy** → Select **Geo-zone-redundant storage (GZRS)**

### Redundancy Comparison

| Feature | LRS | ZRS | GRS | GZRS | RA-GRS | RA-GZRS |
|---------|-----|-----|-----|------|--------|---------|
| **Copies** | 3 | 3 | 6 | 6 | 6 | 6 |
| **Locations** | 1 DC | 3 AZ | 2 regions | 2 regions + AZ | 2 regions | 2 regions + AZ |
| **Zone failure** | ❌ | ✅ | ❌ | ✅ | ❌ | ✅ |
| **Region failure** | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Read from secondary** | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| **Durability (9s)** | 11 | 12 | 16 | 16 | 16 | 16 |
| **Relative cost** | $ | $$ | $$$ | $$$$ | $$$ | $$$$ |

**Exam Scenario:** Application needs protection from datacenter failure but must stay in same region. Answer: ZRS.

### Changing Redundancy

**Supported Conversions:**

| From | To | Method | Downtime |
|------|-----|--------|----------|
| LRS | ZRS, GRS, GZRS | Live migration or manual | None (live) / Yes (manual) |
| ZRS | GZRS | Live migration | None |
| GRS/GZRS | RA-GRS/RA-GZRS | Configuration change | None |
| GRS | LRS, ZRS | Configuration change | None |
| GZRS | GRS, ZRS | Configuration change | None |

**Unsupported Direct Conversions:**
- ZRS → GRS (must go ZRS → GZRS → GRS or ZRS → LRS → GRS)
- LRS → RA-GRS (must go LRS → GRS → RA-GRS)

**Portal Steps:**
1. Navigate to Storage Account → **Data management** → **Redundancy**
2. Select new redundancy option
3. Click **Save**
4. Wait for conversion (can take up to 72 hours)

**Exam Tip:** Conversion to ZRS or GZRS may require live migration request via support ticket for some regions/account types.

### Read Access to Geo-Replicated Storage

RA-GRS and RA-GZRS provide read access to secondary region:

**Secondary Endpoint Format:**
- Primary: `https://storageaccount.blob.core.windows.net`
- Secondary: `https://storageaccount-secondary.blob.core.windows.net`

**Configuration:**
1. Storage account → **Redundancy** → Select **RA-GRS** or **RA-GZRS**
2. Applications use secondary endpoint for read operations
3. Secondary is read-only (write attempts fail)

**Use Cases:**
- Read replicas for high availability
- Disaster recovery testing
- Load distribution for read-heavy workloads
- Geographic read performance optimization

---

## Storage Account Management

### Storage Account Types

| Type | Use Case | Supported Services | Performance |
|------|----------|-------------------|-------------|
| **Standard general-purpose v2** | Most scenarios | Blob, File, Queue, Table | Standard |
| **Premium block blobs** | High transaction rates | Block blobs, append blobs | Premium SSD |
| **Premium file shares** | Enterprise file shares | Azure Files only | Premium SSD |
| **Premium page blobs** | VHD/disk storage | Page blobs (VM disks) | Premium SSD |

**Exam Tip:** Premium storage accounts use SSD, support only LRS or ZRS redundancy, and have higher performance but higher cost.

### Storage Account Properties

#### Performance Tiers

- **Standard:** HDD-based, lower cost, suitable for most workloads
- **Premium:** SSD-based, high IOPS/throughput, low latency, cannot be changed after creation

#### Access Tiers (Blob Storage)

| Tier | Use Case | Storage Cost | Access Cost | Min Duration |
|------|----------|--------------|-------------|--------------|
| **Hot** | Frequently accessed data | Highest | Lowest | None |
| **Cool** | Infrequently accessed, stored 30+ days | Lower | Higher | 30 days |
| **Cold** | Rarely accessed, stored 90+ days | Lower | Higher | 90 days |
| **Archive** | Rarely accessed, stored 180+ days | Lowest | Highest | 180 days |

**Default Access Tier:**
Set at storage account level, applies to new blobs without explicit tier:

1. Storage account → **Configuration**
2. **Blob access tier:** Select Hot or Cool
3. Click **Save**

### Monitoring and Diagnostics

**Metrics Available:**
- Transaction counts
- Ingress/egress bandwidth
- Latency (E2E and server)
- Availability percentage
- Capacity used

**Enabling Diagnostics:**
1. Storage account → **Monitoring** → **Diagnostic settings**
2. Click **+ Add diagnostic setting**
3. Select logs and metrics:
   - **Logs:** Read, Write, Delete operations
   - **Metrics:** Transaction, Capacity
4. Choose destination:
   - Log Analytics workspace
   - Storage account
   - Event Hub
5. Click **Save**

### Soft Delete

Protects blobs and containers from accidental deletion:

**Blob Soft Delete:**
1. Storage account → **Data protection**
2. Under **Recovery**, enable **Enable soft delete for blobs**
3. Set **Retention days** (1-365 days)
4. Click **Save**

**Container Soft Delete:**
1. Storage account → **Data protection**
2. Enable **Enable soft delete for containers**
3. Set **Retention days** (1-365 days)
4. Click **Save**

**Recovering Deleted Items:**
- Portal: Navigate to container → Check **Show deleted blobs/containers**
- Select item → Click **Undelete**

### Versioning

Automatically maintains previous versions of blobs:

1. Storage account → **Data protection**
2. Enable **Enable versioning for blobs**
3. Click **Save**

**Benefits:**
- Protects against accidental overwrites
- Maintains version history
- No additional configuration needed per blob

**Exam Tip:** Versioning and soft delete work together. Versioning handles overwrites, soft delete handles deletions.

---

## AzCopy

AzCopy is a command-line utility for copying data to/from Azure Storage with high performance.

### AzCopy Features

- **High performance:** Parallel transfers, resumable operations
- **Authentication:** Azure AD, SAS tokens, access keys
- **Sync capability:** One-way synchronization
- **Platforms:** Windows, Linux, macOS
- **Services:** Blob, File, Table (limited)

### Installation

**Windows:**
```powershell
# Download and extract
Invoke-WebRequest -Uri "https://aka.ms/downloadazcopy-v10-windows" -OutFile AzCopy.zip
Expand-Archive AzCopy.zip -DestinationPath C:\AzCopy
# Add to PATH
$env:Path += ";C:\AzCopy\azcopy_windows_amd64_10.x.x"
```

**Linux:**
```bash
wget https://aka.ms/downloadazcopy-v10-linux
tar -xvf downloadazcopy-v10-linux
sudo cp ./azcopy_linux_amd64_*/azcopy /usr/bin/
```

**Verification:**
```bash
azcopy --version
```

### Authentication

#### Azure AD (Recommended)

```bash
# Login with Azure AD
azcopy login

# Login with service principal
azcopy login --service-principal --application-id <app-id> --tenant-id <tenant-id>
```

#### SAS Token

```bash
# SAS token appended to URL
azcopy copy "source" "https://account.blob.core.windows.net/container?<sas-token>"
```

### Common Operations

#### Upload Files

**Single file:**
```bash
azcopy copy "C:\local\file.txt" "https://account.blob.core.windows.net/container/file.txt?<sas>"
```

**Directory (recursive):**
```bash
azcopy copy "C:\local\folder" "https://account.blob.core.windows.net/container" --recursive
```

**With pattern matching:**
```bash
azcopy copy "C:\local\*.jpg" "https://account.blob.core.windows.net/container"
```

#### Download Files

**Single blob:**
```bash
azcopy copy "https://account.blob.core.windows.net/container/file.txt?<sas>" "C:\local\file.txt"
```

**Entire container:**
```bash
azcopy copy "https://account.blob.core.windows.net/container?<sas>" "C:\local\folder" --recursive
```

#### Copy Between Storage Accounts

**Blob to blob:**
```bash
azcopy copy "https://source.blob.core.windows.net/container?<src-sas>" \
            "https://dest.blob.core.windows.net/container?<dest-sas>" \
            --recursive
```

#### Synchronize

**One-way sync (local to cloud):**
```bash
azcopy sync "C:\local\folder" "https://account.blob.core.windows.net/container?<sas>" --recursive
```

**Delete destination files not in source:**
```bash
azcopy sync "C:\local\folder" "https://account.blob.core.windows.net/container?<sas>" --recursive --delete-destination=true
```

### AzCopy Options

| Option | Purpose |
|--------|---------|
| `--recursive` | Include subdirectories |
| `--include-pattern` | Include files matching pattern |
| `--exclude-pattern` | Exclude files matching pattern |
| `--block-size-mb` | Upload block size (default 8MB) |
| `--overwrite` | true, false, prompt, ifSourceNewer |
| `--check-md5` | Verify MD5 hash after transfer |
| `--cap-mbps` | Limit bandwidth (MB/s) |
| `--log-level` | INFO, WARNING, ERROR |
| `--dry-run` | Show what would be transferred |

### Performance Tuning

```bash
# Increase concurrency (default 300)
set AZCOPY_CONCURRENCY_VALUE=500

# Increase buffer size
set AZCOPY_BUFFER_GB=4

# Set log location
set AZCOPY_LOG_LOCATION=C:\AzCopy\Logs
```

### Job Management

```bash
# List jobs
azcopy jobs list

# Show job details
azcopy jobs show <job-id>

# Resume failed job
azcopy jobs resume <job-id>

# Remove job
azcopy jobs clean
```

**Exam Scenario:** Need to upload 500GB of files to Azure. Solution: Use AzCopy with --recursive flag and Azure AD authentication for security and performance.

---

## Azure Storage Explorer

Cross-platform desktop application for managing Azure Storage with GUI.

### Features

- **Multi-platform:** Windows, macOS, Linux
- **All storage services:** Blob, File, Queue, Table, Data Lake
- **Multiple accounts:** Manage multiple storage accounts
- **Authentication:** Azure AD, SAS, connection strings, access keys
- **Advanced operations:** Copy, move, delete, ACL management
- **Emulator support:** Azure Storage Emulator/Azurite

### Installation

Download from: https://azure.microsoft.com/features/storage-explorer/

### Connecting to Storage

#### Azure AD Authentication (Recommended)

1. Open Storage Explorer
2. Click **Connect** icon
3. Select **Add an Azure Account**
4. Click **Next** → Sign in with Azure AD
5. Select subscriptions to display

#### SAS URI

1. Click **Connect** icon
2. Select **Blob container**, **File share**, etc.
3. Select **Shared access signature URL (SAS)**
4. Paste SAS URI → Click **Next** → **Connect**

#### Connection String

1. Click **Connect** icon
2. Select **Storage account or service**
3. Select **Connection string**
4. Paste connection string → Click **Next** → **Connect**

### Common Tasks

#### Upload Files

1. Navigate to container or file share
2. Click **Upload** → **Upload Files** or **Upload Folder**
3. Select files/folder
4. Configure:
   - **Blob type:** Block blob, Page blob, Append blob
   - **Access tier:** Hot, Cool, Archive (for blobs)
   - **Upload to folder:** Optional destination path
5. Click **Upload**

#### Download Files

1. Navigate to container/share
2. Select blob(s) or file(s)
3. Click **Download**
4. Select destination folder
5. Click **Select Folder**

#### Copy Between Accounts

1. Right-click blob/container in source account
2. Select **Copy**
3. Navigate to destination account/container
4. Right-click → **Paste**

#### Manage Properties

1. Right-click blob or container
2. Select **Properties**
3. View/edit:
   - Metadata (key-value pairs)
   - Content type
   - Content encoding
   - Cache control
   - Access tier (for blobs)

#### Generate SAS

1. Right-click container, blob, file share, or file
2. Select **Get Shared Access Signature**
3. Configure:
   - **Permissions:** Read, Write, Delete, List
   - **Start time:** Optional
   - **Expiry time:** Required
   - **Allowed protocols:** HTTPS only (recommended)
4. Click **Create**
5. Copy **URL** or **Query string**

### Troubleshooting

**Can't see storage account:**
- Verify subscription is selected in Account Management
- Check Azure AD permissions
- Try refreshing (right-click subscription → Refresh)

**Upload fails:**
- Verify SAS token has write permissions
- Check SAS token hasn't expired
- Verify network connectivity
- Check storage account firewall rules

---

## Object Replication

Asynchronously copies block blobs between storage accounts.

### Use Cases

- **Minimize latency:** Replicate to region closer to users
- **Compute efficiency:** Process data in multiple regions
- **Data distribution:** Copy data for analytics in different regions
- **Cost optimization:** Move data to cheaper storage regions

### Requirements

- **Source & destination:** Must be general-purpose v2 or premium block blob accounts
- **Blob versioning:** Must be enabled on both accounts
- **Change feed:** Must be enabled on source account
- **Same Azure AD tenant:** (unless using cross-tenant replication)

### Configuration

**Portal Steps:**

1. **Enable prerequisites on source account:**
   - Navigate to source storage account
   - **Data protection** → Enable **Versioning**
   - **Data protection** → Enable **Change feed**

2. **Enable prerequisites on destination account:**
   - Navigate to destination account
   - **Data protection** → Enable **Versioning**

3. **Create replication rule:**
   - Navigate to source storage account
   - Select **Data management** → **Object replication**
   - Click **+ Create replication rule**
   - Configure:
     - **Source account:** Current account (pre-filled)
     - **Source container:** Select container
     - **Destination account:** Select destination storage account
     - **Destination container:** Select container (or create new)
     - **Filters:** Optional (prefix match, min blob creation time)
   - Click **Create**

4. **Verification:**
   - Upload blob to source container
   - Wait a few minutes
   - Check destination container for replicated blob

### Replication Policy

```json
{
  "properties": {
    "policyId": "default",
    "sourceAccount": "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/sourceaccount",
    "destinationAccount": "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/destaccount",
    "rules": [
      {
        "ruleId": "rule1",
        "sourceContainer": "source-container",
        "destinationContainer": "dest-container",
        "filters": {
          "prefixMatch": ["folder1/"],
          "minCreationTime": "2024-01-01T00:00:00Z"
        }
      }
    ]
  }
}
```

### Replication Behavior

- **Asynchronous:** Not immediate (typically minutes)
- **New blobs only:** Existing blobs not replicated (must copy manually first)
- **Overwrites replicated:** Changes to source blob replicate
- **Delete not replicated:** Deleting source blob doesn't delete destination
- **Blob versions:** All versions replicated if versioning enabled
- **One-way:** Destination → source replication requires separate rule

**Exam Tip:** Object replication is async and doesn't replicate deletions. For real-time sync or deletion replication, consider other solutions.

---

## Import/Export Service

Physical data transfer service using hard drives shipped to Azure datacenter.

### Use Cases

- **Large data migration:** TBs or PBs of data (faster than network)
- **Bandwidth-limited:** Slow internet connection
- **Initial seeding:** Seed cloud with on-premises data
- **Disaster recovery:** Export data for off-site backup

### Import Job

Transfers data from your drives to Azure Storage.

**Process:**

1. **Prepare drives:**
   - Use WAImportExport tool
   - Copy data to BitLocker-encrypted drives
   - Generate journal files

2. **Create import job:**
   - Azure Portal → **Import/Export jobs**
   - Click **+ Create**
   - Select **Import into Azure**
   - Configure:
     - **Subscription:** Target subscription
     - **Resource group:** Job resource group
     - **Job name:** Descriptive name
     - **Destination:** Storage account and container/file share
     - **Return shipping info:** Your return address
   - Upload journal files
   - Click **Create**

3. **Ship drives:**
   - Ship drives to Azure datacenter address provided
   - Include job ID label

4. **Azure processes:**
   - Receives drives
   - Uploads data to storage account
   - Ships drives back (if requested)

5. **Verify:**
   - Check storage account for imported data
   - Review job status in portal

### Export Job

Transfers data from Azure Storage to your drives.

**Process:**

1. **Create export job:**
   - Azure Portal → **Import/Export jobs**
   - Click **+ Create**
   - Select **Export from Azure**
   - Configure:
     - **Source:** Storage account, container, blob prefix
     - **Number of drives:** How many drives needed
     - **Return shipping info:** Your address
   - Click **Create**

2. **Ship empty drives:**
   - Ship BitLocker-supported drives to Azure
   - Include job ID label

3. **Azure processes:**
   - Receives drives
   - Copies data to drives
   - Encrypts with BitLocker
   - Ships drives to you

4. **Receive drives:**
   - Unlock drives with BitLocker key (in portal)
   - Access exported data

### Requirements

- **Drive type:** SATA II/III HDDs or SSDs (USB adapters not supported in datacenter)
- **Encryption:** BitLocker (Windows drives)
- **File system:** NTFS
- **WAImportExport tool:** For drive preparation
- **Supported regions:** Most Azure regions (check documentation)

**Exam Scenario:** Migrate 50TB of on-premises data to Azure. Internet bandwidth is 100Mbps. Solution: Use Azure Import/Export service (would take ~50 days over internet).

---

## Storage Account Failover

For geo-redundant storage accounts (GRS, GZRS, RA-GRS, RA-GZRS), you can manually fail over to the secondary region.

### When to Failover

- **Primary region outage:** Primary region unavailable for extended period
- **Disaster recovery testing:** Test DR procedures
- **Planned maintenance:** Regional maintenance requiring failover

### Failover Types

#### Customer-Managed Failover (Account Failover)

**Initiated by customer**, makes secondary region the new primary.

**Impact:**
- **DNS update:** Primary endpoint now points to former secondary region
- **Data loss possible:** RPO is ~15 minutes, uncommitted data may be lost
- **Downtime:** During failover (typically 1 hour)
- **LRS conversion:** After failover, account becomes LRS in new primary region
- **Re-enable geo-redundancy:** Must manually change to GRS/GZRS after failover

**Process:**

1. Verify issue warrants failover (primary region down)
2. Navigate to Storage Account → **Data management** → **Geo-replication**
3. Click **Prepare for failover**
4. Review warnings (data loss, LRS conversion)
5. Type storage account name to confirm
6. Click **Failover**
7. Wait for failover to complete (check notifications)
8. **After failover:**
   - Verify data in new primary region
   - Update application configuration to use new primary
   - Re-configure geo-redundancy if needed

**Exam Tip:** After failover, account becomes LRS. Must manually reconfigure to GRS/GZRS for geo-redundancy.

#### Microsoft-Managed Failover

Initiated by Microsoft during large-scale regional disaster. Customers cannot control timing.

### Failback

After issue resolved, you may want to fail back to original region:

1. Re-enable geo-redundancy (LRS → GRS/GZRS)
2. Wait for data to replicate to original region (now secondary)
3. Perform another failover to original region
4. Re-enable geo-redundancy again

**Important:** Requires two failovers (back and forth) and re-enabling geo-redundancy twice.

### Testing Failover

For RA-GRS/RA-GZRS, test failover without actual failover:

1. Configure application to use secondary endpoint
2. Test read operations from secondary
3. Verify application logic handles secondary correctly
4. Switch back to primary endpoint

**Does not require actual failover**, preserves geo-redundancy.

---

## Performance and Scalability

### Scalability Targets

**Standard Storage Account:**

| Resource | Limit |
|----------|-------|
| **Maximum capacity** | 5 PiB |
| **Maximum request rate** | 20,000 requests/second |
| **Maximum ingress** | 60 Gbps (GRS), 120 Gbps (LRS/ZRS) |
| **Maximum egress** | 120 Gbps (GRS), 240 Gbps (LRS/ZRS) |

**Premium Block Blob:**

| Resource | Limit |
|----------|-------|
| **Maximum capacity** | 4 TiB |
| **Maximum IOPS** | 250,000 |
| **Maximum throughput** | 25,000 Mbps |

**Per Blob:**

| Type | Max Size | Max Throughput |
|------|----------|----------------|
| **Block blob** | ~190.7 TiB (50,000 blocks × 4000 MiB) | 60 MiB/s |
| **Page blob** | 8 TiB | 60 MiB/s |
| **Append blob** | ~195 GiB | 60 MiB/s |

### Performance Optimization

**1. Use Premium Storage for high performance:**
- Low latency (single-digit millisecond)
- High IOPS and throughput
- SSD-backed

**2. Increase client-side parallelism:**
- Upload/download multiple blobs simultaneously
- Use multiple connections per blob (block blob only)

**3. Optimize block size:**
- Larger blocks = fewer requests
- Default 4MB, max 100MB per block
- Tune based on file size and network

**4. Use appropriate access tier:**
- Hot tier: Best performance, higher cost
- Cool/Cold tier: Lower performance, lower cost
- Don't use archive for frequently accessed data

**5. Reduce latency with CDN:**
- Azure CDN caches blobs at edge locations
- Reduces latency for geographically distributed users

**6. Use Azure Front Door:**
- Global load balancing and routing
- Improved performance with caching and acceleration

**7. Monitor and optimize:**
- Use Storage Analytics metrics
- Identify bottlenecks (throttling, high latency)
- Adjust redundancy and tiers accordingly

### Throttling

When exceeding scalability targets, requests are throttled (HTTP 503 or 500).

**Resolution:**
- Implement exponential backoff retry logic
- Distribute load across multiple storage accounts
- Use standard storage for non-critical data
- Increase block size to reduce request count

---

## PowerShell & CLI Commands

### Storage Account Management

#### PowerShell

```powershell
# Create storage account
New-AzStorageAccount `
  -ResourceGroupName 'rg-storage' `
  -Name 'storageaccount01' `
  -Location 'eastus' `
  -SkuName 'Standard_GRS' `
  -Kind 'StorageV2' `
  -AccessTier 'Hot' `
  -EnableHttpsTrafficOnly $true `
  -MinimumTlsVersion 'TLS1_2'

# Change redundancy
Set-AzStorageAccount `
  -ResourceGroupName 'rg-storage' `
  -Name 'storageaccount01' `
  -SkuName 'Standard_GZRS'

# Enable RA-GRS
Set-AzStorageAccount `
  -ResourceGroupName 'rg-storage' `
  -Name 'storageaccount01' `
  -SkuName 'Standard_RAGRS'

# Get storage account properties
Get-AzStorageAccount `
  -ResourceGroupName 'rg-storage' `
  -Name 'storageaccount01'

# List storage accounts
Get-AzStorageAccount -ResourceGroupName 'rg-storage'

# Change access tier
Set-AzStorageAccount `
  -ResourceGroupName 'rg-storage' `
  -Name 'storageaccount01' `
  -AccessTier 'Cool'

# Get storage account keys
Get-AzStorageAccountKey `
  -ResourceGroupName 'rg-storage' `
  -Name 'storageaccount01'

# Create storage context
$ctx = New-AzStorageContext `
  -StorageAccountName 'storageaccount01' `
  -StorageAccountKey '<key>'
```

#### Azure CLI

```bash
# Create storage account
az storage account create \
  --name storageaccount01 \
  --resource-group rg-storage \
  --location eastus \
  --sku Standard_GRS \
  --kind StorageV2 \
  --access-tier Hot \
  --https-only true \
  --min-tls-version TLS1_2

# Change redundancy
az storage account update \
  --name storageaccount01 \
  --resource-group rg-storage \
  --sku Standard_GZRS

# Enable RA-GRS
az storage account update \
  --name storageaccount01 \
  --resource-group rg-storage \
  --sku Standard_RAGRS

# Get storage account properties
az storage account show \
  --name storageaccount01 \
  --resource-group rg-storage

# List storage accounts
az storage account list \
  --resource-group rg-storage \
  --output table

# Change access tier
az storage account update \
  --name storageaccount01 \
  --resource-group rg-storage \
  --access-tier Cool

# Get storage account keys
az storage account keys list \
  --account-name storageaccount01 \
  --resource-group rg-storage \
  --output table
```

### Blob Operations

#### PowerShell

```powershell
# Upload blob
Set-AzStorageBlobContent `
  -File 'C:\files\document.pdf' `
  -Container 'documents' `
  -Blob 'document.pdf' `
  -Context $ctx `
  -Properties @{'ContentType'='application/pdf'}

# Download blob
Get-AzStorageBlobContent `
  -Container 'documents' `
  -Blob 'document.pdf' `
  -Destination 'C:\downloads\document.pdf' `
  -Context $ctx

# List blobs
Get-AzStorageBlob -Container 'documents' -Context $ctx

# Copy blob between containers
Start-AzStorageBlobCopy `
  -SrcContainer 'source' `
  -SrcBlob 'file.txt' `
  -DestContainer 'destination' `
  -DestBlob 'file.txt' `
  -Context $ctx

# Get copy status
Get-AzStorageBlobCopyState `
  -Container 'destination' `
  -Blob 'file.txt' `
  -Context $ctx

# Change blob tier
Set-AzStorageBlobContent `
  -Container 'documents' `
  -Blob 'archive.zip' `
  -StandardBlobTier 'Archive' `
  -Context $ctx

# Delete blob
Remove-AzStorageBlob `
  -Container 'documents' `
  -Blob 'document.pdf' `
  -Context $ctx
```

#### Azure CLI

```bash
# Upload blob
az storage blob upload \
  --account-name storageaccount01 \
  --container-name documents \
  --name document.pdf \
  --file C:\files\document.pdf \
  --content-type application/pdf

# Download blob
az storage blob download \
  --account-name storageaccount01 \
  --container-name documents \
  --name document.pdf \
  --file C:\downloads\document.pdf

# List blobs
az storage blob list \
  --account-name storageaccount01 \
  --container-name documents \
  --output table

# Copy blob between containers
az storage blob copy start \
  --account-name storageaccount01 \
  --destination-container destination \
  --destination-blob file.txt \
  --source-container source \
  --source-blob file.txt

# Get copy status
az storage blob show \
  --account-name storageaccount01 \
  --container-name destination \
  --name file.txt \
  --query 'properties.copy'

# Change blob tier
az storage blob set-tier \
  --account-name storageaccount01 \
  --container-name documents \
  --name archive.zip \
  --tier Archive

# Delete blob
az storage blob delete \
  --account-name storageaccount01 \
  --container-name documents \
  --name document.pdf
```

### Soft Delete and Versioning

#### PowerShell

```powershell
# Enable blob soft delete
Enable-AzStorageBlobDeleteRetentionPolicy `
  -ResourceGroupName 'rg-storage' `
  -StorageAccountName 'storageaccount01' `
  -RetentionDays 7

# Enable container soft delete
Enable-AzStorageContainerDeleteRetentionPolicy `
  -ResourceGroupName 'rg-storage' `
  -StorageAccountName 'storageaccount01' `
  -RetentionDays 7

# Enable versioning
Update-AzStorageBlobServiceProperty `
  -ResourceGroupName 'rg-storage' `
  -StorageAccountName 'storageaccount01' `
  -IsVersioningEnabled $true

# List deleted blobs
Get-AzStorageBlob -Container 'documents' -Context $ctx -IncludeDeleted

# Restore deleted blob
Restore-AzStorageBlob `
  -Container 'documents' `
  -Blob 'document.pdf' `
  -Context $ctx
```

#### Azure CLI

```bash
# Enable blob soft delete
az storage account blob-service-properties update \
  --account-name storageaccount01 \
  --resource-group rg-storage \
  --enable-delete-retention true \
  --delete-retention-days 7

# Enable container soft delete
az storage account blob-service-properties update \
  --account-name storageaccount01 \
  --resource-group rg-storage \
  --enable-container-delete-retention true \
  --container-delete-retention-days 7

# Enable versioning
az storage account blob-service-properties update \
  --account-name storageaccount01 \
  --resource-group rg-storage \
  --enable-versioning true

# List deleted blobs
az storage blob list \
  --account-name storageaccount01 \
  --container-name documents \
  --include d \
  --output table

# Restore deleted blob
az storage blob undelete \
  --account-name storageaccount01 \
  --container-name documents \
  --name document.pdf
```

### Object Replication

#### PowerShell

```powershell
# Create replication policy
$srcAccount = Get-AzStorageAccount -ResourceGroupName 'rg-source' -Name 'sourceaccount'
$destAccount = Get-AzStorageAccount -ResourceGroupName 'rg-dest' -Name 'destaccount'

$rule = New-AzStorageObjectReplicationPolicyRule `
  -SourceContainer 'source-container' `
  -DestinationContainer 'dest-container'

Set-AzStorageObjectReplicationPolicy `
  -ResourceGroupName 'rg-source' `
  -StorageAccountName 'sourceaccount' `
  -PolicyId 'default' `
  -SourceAccount $srcAccount.Id `
  -DestinationAccount $destAccount.Id `
  -Rule $rule

# Get replication status
Get-AzStorageObjectReplicationPolicy `
  -ResourceGroupName 'rg-source' `
  -StorageAccountName 'sourceaccount'
```

#### Azure CLI

```bash
# Create replication policy (requires JSON file)
az storage account or-policy create \
  --account-name sourceaccount \
  --resource-group rg-source \
  --source-account sourceaccount \
  --destination-account destaccount \
  --source-container source-container \
  --destination-container dest-container

# List replication policies
az storage account or-policy list \
  --account-name sourceaccount \
  --resource-group rg-source

# Show replication policy
az storage account or-policy show \
  --account-name sourceaccount \
  --resource-group rg-source \
  --policy-id <policy-id>
```

### Storage Account Failover

#### PowerShell

```powershell
# Initiate failover
Invoke-AzStorageAccountFailover `
  -ResourceGroupName 'rg-storage' `
  -Name 'storageaccount01' `
  -Force

# Check failover status
Get-AzStorageAccount `
  -ResourceGroupName 'rg-storage' `
  -Name 'storageaccount01' | Select-Object StatusOfPrimary, StatusOfSecondary
```

#### Azure CLI

```bash
# Initiate failover
az storage account failover \
  --name storageaccount01 \
  --resource-group rg-storage \
  --yes

# Check failover status
az storage account show \
  --name storageaccount01 \
  --resource-group rg-storage \
  --query '{primary:statusOfPrimary, secondary:statusOfSecondary}'
```

---

## Exam Tips

### Key Concepts for AZ-104

1. **Redundancy Tiers:**
   - **LRS:** 3 copies in 1 datacenter (11 nines durability)
   - **ZRS:** 3 copies across 3 zones (12 nines durability)
   - **GRS:** 6 copies across 2 regions (16 nines durability)
   - **GZRS:** ZRS in primary + LRS in secondary (16 nines durability)
   - **Exam Tip:** More nines = more durability, more cost

2. **Redundancy Changes:**
   - Can change between most types (some require live migration request)
   - ZRS → GRS requires two-step process (ZRS → GZRS → GRS or ZRS → LRS → GRS)
   - Premium accounts support only LRS or ZRS
   - **Exam Scenario:** Converting GRS to LRS has no downtime

3. **AzCopy Usage:**
   - Use `azcopy copy` for upload/download/copy
   - Use `azcopy sync` for one-way synchronization
   - Supports Azure AD authentication (most secure)
   - Resumable transfers with job management
   - **Exam Scenario:** Transfer TBs of data between accounts → Use AzCopy

4. **Object Replication:**
   - Requires versioning and change feed enabled
   - Asynchronous (not immediate)
   - Doesn't replicate deletions
   - New blobs only (existing blobs require manual copy first)
   - **Exam Trap:** Object replication is NOT for disaster recovery (use GRS)

5. **Storage Account Failover:**
   - Only for GRS, GZRS, RA-GRS, RA-GZRS accounts
   - Results in LRS after failover (must re-enable geo-redundancy)
   - Potential data loss (RPO ~15 minutes)
   - Requires manual intervention (not automatic)
   - **Exam Scenario:** Primary region down → Initiate account failover

6. **Import/Export Service:**
   - For large-scale data transfer (TBs/PBs)
   - Physical disk shipping to/from Azure
   - Faster and cheaper than network for massive datasets
   - Supports both import (to Azure) and export (from Azure)
   - **Exam Scenario:** 50TB migration with slow internet → Use Import/Export

7. **Soft Delete:**
   - Retention period: 1-365 days
   - Available for blobs and containers
   - Protects against accidental deletion
   - Must be explicitly enabled
   - **Exam Tip:** Soft delete + versioning = comprehensive protection

8. **Storage Explorer:**
   - GUI tool for managing storage
   - Supports all authentication methods
   - Works with multiple storage accounts simultaneously
   - Useful for one-time operations and troubleshooting
   - **Exam Scenario:** Non-technical user needs to upload files → Use Storage Explorer

### Common Exam Scenarios

**Scenario 1: High Availability Across Zones**
- **Question:** Application needs protection from datacenter failure in same region
- **Answer:** Use Zone-redundant storage (ZRS)
- **Why:** ZRS replicates across 3 availability zones in same region

**Scenario 2: Disaster Recovery with Read Access**
- **Question:** Application needs DR and ability to read from secondary region
- **Answer:** Use Read-access geo-redundant storage (RA-GRS) or RA-GZRS
- **Why:** RA-GRS/RA-GZRS provides geo-replication with read access to secondary

**Scenario 3: Migrate 100TB to Azure**
- **Question:** On-premises has 100TB data, 500 Mbps connection. Best migration method?
- **Answer:** Azure Import/Export service
- **Why:** Would take ~18 days over network vs. ~1 week with disk shipping

**Scenario 4: Copy Data Between Accounts**
- **Question:** Copy 10TB from storage account A to storage account B
- **Answer:** Use AzCopy with service-to-service copy
- **Why:** AzCopy is fastest, most efficient for large-scale copies

**Scenario 5: Protect Against Accidental Deletion**
- **Question:** Prevent accidental blob deletion, allow recovery for 30 days
- **Answer:** Enable blob soft delete with 30-day retention
- **Why:** Soft delete maintains deleted blobs for specified retention period

**Scenario 6: Convert LRS to GZRS**
- **Question:** Need to convert standard LRS account to GZRS for better availability
- **Answer:** Change redundancy setting to GZRS (may require support ticket for live migration)
- **Why:** Direct conversion supported, but some regions/types need migration request

**Scenario 7: Primary Region Failure**
- **Question:** Storage account uses GRS, primary region is down for 4 hours
- **Answer:** Initiate customer-managed account failover to secondary region
- **Why:** GRS supports manual failover during regional outage

**Scenario 8: Distribute Data Globally**
- **Question:** Need data automatically replicated to multiple regions for low latency
- **Answer:** Use object replication between multiple storage accounts
- **Why:** Object replication copies blobs asynchronously to other regions

---

## Common Scenarios & Troubleshooting

### Scenario: AzCopy Upload Fails

**Symptoms:**
- Transfer stops mid-upload
- Authentication errors
- Network timeout errors

**Troubleshooting Steps:**
1. **Check authentication:** Verify SAS token or Azure AD login
2. **Check SAS permissions:** Ensure write permission granted
3. **Check SAS expiry:** Token may have expired
4. **Check network:** Verify internet connectivity
5. **Check firewall:** Storage account may have network restrictions
6. **Review logs:** Check AzCopy logs for detailed errors

**Resolution:**
```bash
# Re-authenticate with Azure AD
azcopy login

# Resume failed job
azcopy jobs list
azcopy jobs resume <job-id>

# Or restart with fresh SAS token
azcopy copy "source" "destination?<new-sas>" --recursive
```

### Scenario: Object Replication Not Working

**Symptoms:**
- Blobs not appearing in destination container
- Replication delay longer than expected

**Troubleshooting Steps:**
1. **Verify prerequisites:**
   - Versioning enabled on both accounts
   - Change feed enabled on source account
2. **Check replication policy:** Verify rule is active and correctly configured
3. **Check filters:** Prefix or time filters may exclude blobs
4. **Check new blobs only:** Existing blobs aren't replicated
5. **Wait longer:** Replication is asynchronous (can take 10+ minutes)

**Resolution:**
```powershell
# Verify versioning
Get-AzStorageBlobServiceProperty -ResourceGroupName 'rg-storage' -StorageAccountName 'sourceaccount'

# Check replication policy
Get-AzStorageObjectReplicationPolicy -ResourceGroupName 'rg-storage' -StorageAccountName 'sourceaccount'

# Manually copy existing blobs
$ctx = New-AzStorageContext -StorageAccountName 'sourceaccount' -StorageAccountKey $key
Get-AzStorageBlob -Container 'source' -Context $ctx | ForEach-Object {
    Start-AzStorageBlobCopy -SrcContainer 'source' -SrcBlob $_.Name `
        -DestContainer 'dest' -DestBlob $_.Name -Context $ctx
}
```

### Scenario: Cannot Change Redundancy

**Symptoms:**
- Redundancy change option grayed out
- Error message when attempting to change

**Troubleshooting Steps:**
1. **Check account type:** Premium accounts support only LRS/ZRS
2. **Check conversion path:** Some conversions require two steps
3. **Check region support:** Not all regions support all redundancy types
4. **Check pending operations:** Account may have active failover or migration
5. **Check account status:** Account must be in healthy state

**Resolution:**
```powershell
# Check current redundancy
$sa = Get-AzStorageAccount -ResourceGroupName 'rg-storage' -Name 'storageaccount01'
$sa.Sku.Name

# For ZRS → GRS, use two-step conversion
# Step 1: ZRS → GZRS
Set-AzStorageAccount -ResourceGroupName 'rg-storage' -Name 'storageaccount01' -SkuName 'Standard_GZRS'

# Wait for conversion to complete (check portal or poll status)
# Step 2: GZRS → GRS
Set-AzStorageAccount -ResourceGroupName 'rg-storage' -Name 'storageaccount01' -SkuName 'Standard_GRS'
```

### Scenario: Failover Fails or Stalls

**Symptoms:**
- Failover doesn't complete
- Storage account shows "Failover in progress" for extended time

**Troubleshooting Steps:**
1. **Wait longer:** Failover can take 1+ hours
2. **Check geo-replication status:** Must be "Available" before failover
3. **Check account type:** Only GRS/GZRS/RA-GRS/RA-GZRS support failover
4. **Check ongoing operations:** Large data transfers may delay failover
5. **Contact support:** If stalled for 4+ hours

**Resolution:**
```powershell
# Check geo-replication status
$sa = Get-AzStorageAccount -ResourceGroupName 'rg-storage' -Name 'storageaccount01'
$sa.GeoReplicationStats

# If status shows "Available", can proceed with failover
Invoke-AzStorageAccountFailover -ResourceGroupName 'rg-storage' -Name 'storageaccount01' -Force
```

### Scenario: Soft-Deleted Blobs Not Showing

**Symptoms:**
- Deleted blobs not visible in portal or Storage Explorer
- Cannot restore deleted blobs

**Troubleshooting Steps:**
1. **Check soft delete enabled:** May not have been enabled before deletion
2. **Check retention period:** Blob may have been deleted before soft delete enabled
3. **Check retention expired:** Blobs deleted >retention days ago are permanently deleted
4. **Check view settings:** Portal may not show deleted blobs by default

**Resolution:**
```powershell
# Verify soft delete configuration
Get-AzStorageBlobServiceProperty -ResourceGroupName 'rg-storage' -StorageAccountName 'storageaccount01'

# List deleted blobs
Get-AzStorageBlob -Container 'documents' -Context $ctx -IncludeDeleted

# Restore specific blob
Restore-AzStorageBlob -Container 'documents' -Blob 'deleted-file.pdf' -Context $ctx
```

**Portal:**
1. Navigate to container
2. Check **Show deleted blobs** toggle
3. Select deleted blob
4. Click **Undelete**

---

## Related Topics

### Linked Study Materials

- **[Secure Storage](./Secure%20Storage.md):** Security, SAS tokens, encryption, network rules
- **[Configure Azure Files and Blob Storage](./Configure%20Azure%20Files%20and%20Blob%20Storage.md):** File shares, blob tiers, lifecycle management
- **[Implement Backup and Recovery](../Monitor%20and%20Backup%20Azure%20Resources/Implement%20Backup%20and%20Recovery.md):** Azure Backup for storage
- **[Monitor Resources with Azure Monitor](../Monitor%20and%20Backup%20Azure%20Resources/Monitor%20Resources%20with%20Azure%20Monitor.md):** Storage monitoring and alerts

### Azure Services Integration

- **Azure Backup:** Backup Azure Files shares and blob containers
- **Azure Site Recovery:** Replicate VMs using storage accounts
- **Azure Data Factory:** Data movement and transformation with storage
- **Azure Synapse Analytics:** Analytics on data in storage accounts
- **Azure CDN:** Cache blob content at edge locations

### Advanced Topics (Beyond AZ-104)

- **Azure Data Lake Storage Gen2:** Hierarchical namespace for big data analytics
- **Azure HPC Cache:** High-performance cache for file-based workloads
- **Azure NetApp Files:** Enterprise-grade file storage service
- **Storage Actions:** Manage data lifecycle at scale across accounts
- **Azure Data Box:** Hardware-based data transfer solutions

### Documentation Links

- [Storage Redundancy Overview](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)
- [AzCopy Documentation](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-v10)
- [Storage Explorer](https://learn.microsoft.com/en-us/azure/vs-azure-tools-storage-manage-with-storage-explorer)
- [Object Replication](https://learn.microsoft.com/en-us/azure/storage/blobs/object-replication-overview)
- [Import/Export Service](https://learn.microsoft.com/en-us/azure/import-export/storage-import-export-service)
- [Account Failover](https://learn.microsoft.com/en-us/azure/storage/common/storage-disaster-recovery-guidance)

---

**Study Checkpoint:** You should now understand storage redundancy options, data transfer tools (AzCopy, Storage Explorer), object replication, Import/Export service, and account failover procedures. Practice changing redundancy, using AzCopy commands, and configuring object replication.

**Next Steps:** Review [Configure Azure Files and Blob Storage](./Configure%20Azure%20Files%20and%20Blob%20Storage.md) for service-specific features and [Secure Storage](./Secure%20Storage.md) for security configurations.
