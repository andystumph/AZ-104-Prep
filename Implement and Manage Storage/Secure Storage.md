# Secure Storage

## Table of Contents
1. [Overview](#overview)
2. [Storage Account Access Keys](#storage-account-access-keys)
3. [Shared Access Signatures (SAS)](#shared-access-signatures-sas)
4. [Azure AD Authentication](#azure-ad-authentication)
5. [Storage Firewalls and Virtual Networks](#storage-firewalls-and-virtual-networks)
6. [Private Endpoints](#private-endpoints)
7. [Encryption at Rest](#encryption-at-rest)
8. [Encryption in Transit](#encryption-in-transit)
9. [Advanced Threat Protection](#advanced-threat-protection)
10. [PowerShell & CLI Commands](#powershell--cli-commands)
11. [Exam Tips](#exam-tips)
12. [Common Scenarios & Troubleshooting](#common-scenarios--troubleshooting)
13. [Related Topics](#related-topics)

---

## Overview

Azure Storage security encompasses multiple layers of protection to safeguard data at rest, in transit, and during access. Understanding these security mechanisms is critical for the AZ-104 exam and for implementing secure storage solutions in production.

### Storage Security Layers

| Layer | Technologies | Purpose |
|-------|-------------|---------|
| **Identity & Access** | Azure AD, RBAC, SAS, Access Keys | Control who can access storage |
| **Network Security** | Firewalls, Virtual Networks, Private Endpoints | Control where access originates |
| **Data Protection** | Encryption at rest, Encryption in transit, TLS 1.2+ | Protect data confidentiality |
| **Threat Detection** | Microsoft Defender for Storage | Detect and respond to threats |
| **Compliance** | Immutability, Legal Hold, Soft Delete | Meet regulatory requirements |

**Exam Insight:** Always prefer Azure AD authentication over access keys. Access keys provide full account access and should be protected like passwords.

---

## Storage Account Access Keys

Storage account access keys (also called shared keys) provide full access to all data and operations in a storage account.

### Key Characteristics

- **Two keys:** key1 and key2 for zero-downtime rotation
- **Full access:** Complete control over all data and services (Blob, File, Queue, Table)
- **512-bit keys:** Automatically generated
- **No expiration:** Keys never expire unless manually regenerated
- **High risk:** Compromised keys give attacker full storage account access

**Security Risk:** Anyone with a storage account key can read, write, and delete all data. Treat access keys like root passwords.

### When to Use Access Keys

| Use Case | Recommended? | Alternative |
|----------|--------------|-------------|
| Azure services with managed identities | ❌ No | Use Azure AD and managed identities |
| Application authentication | ❌ No | Use SAS tokens with limited permissions |
| Legacy applications | ⚠️ Temporary only | Migrate to Azure AD or SAS |
| Development/testing | ⚠️ Limited use | Use time-limited SAS tokens |
| Key-based protocols (SMB) | ✅ Yes | No alternative for certain scenarios |

### Rotating Access Keys

**Zero-Downtime Rotation Process:**

1. **Update applications to use key2**
2. **Regenerate key1** (applications still use key2)
3. **Update applications to use new key1**
4. **Regenerate key2** (applications now use key1)

**Portal Steps:**
1. Navigate to Storage Account → **Security + networking** → **Access keys**
2. Note current key in use (key1 or key2)
3. Click **Rotate key** on the key NOT currently in use
4. Update application configuration
5. Test application connectivity
6. Rotate the other key

**Exam Scenario:** Application loses storage access after key rotation. Troubleshoot: Check which key the application uses and whether it was regenerated.

### Disabling Shared Key Access

For enhanced security, you can prevent all Shared Key authorization:

**Portal Steps:**
1. Navigate to Storage Account → **Configuration**
2. Under **Allow storage account key access**, select **Disabled**
3. Click **Save**

**Impact:**
- ✅ Forces all access to use Azure AD or SAS
- ✅ Prevents inadvertent key exposure
- ❌ Breaks applications still using access keys
- ❌ Some features require keys (e.g., File REST API with keys)

**Exam Tip:** Disabling shared key access is a security best practice but requires planning to migrate applications first.

---

## Shared Access Signatures (SAS)

Shared Access Signatures provide delegated, time-limited, granular access to storage resources without exposing account keys.

### SAS Types

#### 1. User Delegation SAS (Recommended)
- **Secured by:** Azure AD credentials
- **Applies to:** Blob storage only
- **Benefits:** Can be revoked via Azure AD, no account key exposure
- **Use case:** External users/applications need temporary blob access

#### 2. Service SAS
- **Secured by:** Storage account key
- **Applies to:** Single service (Blob, File, Queue, or Table)
- **Benefits:** Granular permissions per resource
- **Use case:** Grant access to specific containers or blobs

#### 3. Account SAS
- **Secured by:** Storage account key
- **Applies to:** One or more services
- **Benefits:** Access multiple services with one token
- **Use case:** Application needs access to blobs and tables

### SAS Token Components

A SAS URI combines the resource URI with a SAS token:

```
https://storageaccount.blob.core.windows.net/container/blob.txt?
sp=r&
st=2024-01-01T00:00:00Z&
se=2024-01-02T00:00:00Z&
spr=https&
sv=2021-06-08&
sr=b&
sig=<signature>
```

| Parameter | Name | Description |
|-----------|------|-------------|
| `sp` | Signed Permissions | r (read), w (write), d (delete), l (list), a (add), c (create) |
| `st` | Start Time | When SAS becomes valid (optional, defaults to now) |
| `se` | Expiry Time | When SAS expires (required) |
| `spr` | Protocol | https or http,https |
| `sv` | API Version | Storage service version |
| `sr` | Resource Type | b (blob), c (container), bs (blob service) |
| `sig` | Signature | HMAC signature computed with account key |

### Creating a SAS Token

#### User Delegation SAS (Portal)

1. Navigate to Storage Account → **Containers**
2. Select container or blob
3. Click **Generate SAS**
4. Configure:
   - **Signing method:** User delegation key (Azure AD)
   - **Signing key:** Authenticate with Azure AD
   - **Permissions:** Select r, w, d, l as needed
   - **Start time:** Optional (defaults to now)
   - **Expiry time:** Required (e.g., 8 hours from now)
   - **Allowed IP addresses:** Optional (e.g., 203.0.113.0/24)
   - **Allowed protocols:** HTTPS only (recommended)
5. Click **Generate SAS token and URL**
6. Copy the **Blob SAS URL** (includes token)

#### Service SAS (Portal)

1. Navigate to Storage Account → **Shared access signature**
2. Configure:
   - **Allowed services:** Blob, File, Queue, Table
   - **Allowed resource types:** Service, Container, Object
   - **Allowed permissions:** Read, Write, Delete, List, etc.
   - **Start and expiry date/time**
   - **Allowed IP addresses:** Optional
   - **Allowed protocols:** HTTPS only
3. Click **Generate SAS and connection string**
4. Copy the **SAS token** (starts with `?`)

### SAS Best Practices

1. **Use User Delegation SAS** for blobs whenever possible (secured with Azure AD)
2. **Short expiry times:** Hours or days, not months or years
3. **Minimal permissions:** Grant only required permissions (principle of least privilege)
4. **IP restrictions:** Limit to known client IP addresses
5. **HTTPS only:** Never allow HTTP for SAS tokens
6. **Stored Access Policy:** Use with service SAS for revocation capability
7. **Monitor usage:** Check Storage Analytics logs for SAS usage

**Exam Scenario:** External partner needs read access to a specific container for 24 hours. Solution: Generate User Delegation SAS with read-only permission, 24-hour expiry, and partner's IP address allowlist.

### Stored Access Policies

Stored Access Policies allow you to change or revoke service SAS tokens after they're issued.

**Creating a Stored Access Policy:**

1. Navigate to Storage Account → **Containers** → Select container
2. Click **Access policy**
3. Under **Stored access policies**, click **+ Add policy**
4. Configure:
   - **Identifier:** Policy name (e.g., "external-read")
   - **Permissions:** Read, List
   - **Start time:** Optional
   - **Expiry time:** When policy expires
5. Click **OK** then **Save**

**Using the Policy in SAS:**
When generating service SAS, reference the policy identifier instead of specifying permissions and expiry in the token.

**Revoking Access:**
1. Edit the stored access policy
2. Set expiry time to the past
3. All SAS tokens using this policy become invalid immediately

**Exam Tip:** Stored Access Policies only work with service SAS, not account SAS or user delegation SAS.

---

## Azure AD Authentication

Azure Active Directory (Azure AD) authentication provides identity-based access control for Blob and Queue storage using OAuth 2.0 tokens.

### Benefits of Azure AD Authentication

| Benefit | Description |
|---------|-------------|
| **No credential storage** | Applications don't need to store keys or SAS tokens |
| **Centralized access control** | Manage access via Azure RBAC |
| **Conditional Access** | Enforce MFA, device compliance, location restrictions |
| **Managed Identities** | Azure services can authenticate without credentials |
| **Auditing** | Track access by user identity in Azure AD logs |
| **Fine-grained permissions** | Built-in and custom RBAC roles |

### Azure AD Storage Roles

| Role | Permissions | Scope |
|------|------------|-------|
| **Storage Blob Data Owner** | Full access to containers and blobs, set ACLs (ADLS Gen2) | Container, Storage Account |
| **Storage Blob Data Contributor** | Read, write, delete blobs and containers | Container, Storage Account |
| **Storage Blob Data Reader** | Read and list blobs and containers | Container, Storage Account |
| **Storage Queue Data Contributor** | Read, write, delete queue messages | Queue, Storage Account |
| **Storage Queue Data Reader** | Read and peek queue messages | Queue, Storage Account |
| **Storage Table Data Contributor** | Read, write, delete table data | Table, Storage Account |
| **Storage Table Data Reader** | Read table data | Table, Storage Account |

**Exam Important:** These are DATA PLANE roles for accessing storage contents. Don't confuse with MANAGEMENT PLANE roles like Storage Account Contributor (manages the account itself, not the data).

### Assigning Azure AD Roles

**Portal Steps:**
1. Navigate to Storage Account → **Containers** → Select container
2. Click **Access Control (IAM)**
3. Click **+ Add** → **Add role assignment**
4. Select role (e.g., Storage Blob Data Reader)
5. Click **Next**
6. Click **+ Select members**
7. Search for and select user, group, or managed identity
8. Click **Review + assign**

### Managed Identities for Storage Access

Applications running in Azure can use managed identities for passwordless authentication:

**Supported Azure Services:**
- Azure VMs
- Azure App Service
- Azure Functions
- Azure Container Instances
- Azure Kubernetes Service
- Azure Logic Apps
- Azure Data Factory

**Configuration Example (App Service):**

1. Enable managed identity on App Service:
   - Navigate to App Service → **Identity**
   - Enable **System assigned** identity
   - Click **Save**

2. Grant storage access:
   - Navigate to Storage Account → **Access Control (IAM)**
   - Assign **Storage Blob Data Contributor** role to the App Service's managed identity

3. Application code (C# example):
```csharp
using Azure.Identity;
using Azure.Storage.Blobs;

// Managed identity authentication (no keys!)
var blobServiceClient = new BlobServiceClient(
    new Uri("https://storageaccount.blob.core.windows.net"),
    new DefaultAzureCredential());
```

**Exam Scenario:** Azure Function needs to read from Blob storage. Solution: Enable managed identity on Function App, assign Storage Blob Data Reader role, use DefaultAzureCredential in code.

---

## Storage Firewalls and Virtual Networks

Storage account firewalls restrict access to specific networks, IP addresses, or Azure services.

### Network Rule Types

| Rule Type | Purpose | Use Case |
|-----------|---------|----------|
| **Virtual Networks** | Allow specific VNets/subnets | Azure VMs and services in your VNets |
| **IP address ranges** | Allow specific public IPs | On-premises networks, remote offices |
| **Azure services** | Allow trusted Microsoft services | Azure Backup, Azure Site Recovery, Log Analytics |

### Default Network Access

**Two options:**
1. **Enabled from all networks** (default): Storage account is publicly accessible from internet
2. **Enabled from selected virtual networks and IP addresses**: Deny by default, allow specific networks

**Exam Tip:** Changing to "selected networks" immediately blocks all access except configured exceptions. Plan exceptions before applying.

### Configuring Network Rules

**Portal Steps:**
1. Navigate to Storage Account → **Security + networking** → **Networking**
2. Under **Firewalls and virtual networks**, select:
   - **Enabled from all networks** (public access)
   - **Enabled from selected virtual networks and IP addresses** (restricted)
3. If selected networks:
   
   **Add Virtual Network:**
   - Click **+ Add existing virtual network**
   - Select subscription, VNet, and subnet
   - Click **Add** (may take 15 minutes to apply)

   **Add IP Address Range:**
   - Under **Firewall**, enter IP or CIDR range (e.g., 203.0.113.0/24)
   - Click **Save**

   **Allow Azure Services:**
   - Check **Allow Azure services on the trusted services list to access this storage account**
   - Specify exceptions (e.g., Logging, Metrics)

4. Click **Save**

### Service Endpoints vs. Private Endpoints

| Feature | Service Endpoint | Private Endpoint |
|---------|------------------|------------------|
| **IP address** | Public IP | Private IP (from VNet) |
| **Routing** | Via Microsoft backbone | Entirely within VNet |
| **NSG support** | Limited | Full NSG support |
| **DNS** | Public DNS | Private DNS zone |
| **Cost** | Free | Charged per endpoint per hour + data |
| **Use case** | Basic VNet restriction | Highest security, on-prem access |

### Service Endpoints

Enable service endpoint on subnet:

1. Navigate to **Virtual Network** → **Subnets** → Select subnet
2. Under **Service endpoints**, click **Add**
3. Select **Microsoft.Storage**
4. Click **Save**

Add subnet to storage firewall (as shown above in network rules).

**How it works:** Traffic from VNet to storage uses Azure backbone network, appears to storage account as originating from VNet, not public IP.

---

## Private Endpoints

Private endpoints provide a private IP address from your VNet for the storage account, enabling fully private connectivity.

### Private Endpoint Benefits

1. **Private IP address:** Storage accessible via VNet IP (e.g., 10.0.1.4)
2. **No public internet:** Traffic never leaves Microsoft network
3. **On-premises access:** Via VPN or ExpressRoute to VNet
4. **Network security:** Full NSG and route table support
5. **DNS integration:** Private DNS zone or custom DNS configuration

### Creating a Private Endpoint

**Portal Steps:**
1. Navigate to Storage Account → **Security + networking** → **Networking**
2. Click **Private endpoint connections** tab
3. Click **+ Private endpoint**
4. Configure **Basics:**
   - **Name:** pe-storageaccount
   - **Region:** Must match VNet region
5. Configure **Resource:**
   - **Resource type:** Microsoft.Storage/storageAccounts
   - **Resource:** Select your storage account
   - **Target sub-resource:** blob, file, table, queue, or web
6. Configure **Virtual Network:**
   - **Virtual network:** Select VNet
   - **Subnet:** Select subnet (recommended: dedicated subnet for private endpoints)
   - **Private IP configuration:** Dynamic (recommended) or Static
7. Configure **DNS:**
   - **Integrate with private DNS zone:** Yes (recommended)
   - **Private DNS zone:** Creates new or uses existing (e.g., privatelink.blob.core.windows.net)
8. Click **Review + create** → **Create**

### Private Endpoint DNS Resolution

Private endpoint requires DNS configuration for name resolution:

**Automatic (Integrated Private DNS Zone):**
- Azure creates Private DNS zone: `privatelink.blob.core.windows.net`
- A record: `storageaccount.privatelink.blob.core.windows.net` → `10.0.1.4`
- Public DNS returns CNAME: `storageaccount.blob.core.windows.net` → `storageaccount.privatelink.blob.core.windows.net`

**Manual (Custom DNS):**
1. Create conditional forwarder to Azure DNS (168.63.129.16)
2. Or create A record in your DNS server

**Validation:**
```powershell
# From VM in VNet
nslookup storageaccount.blob.core.windows.net
# Should return private IP (e.g., 10.0.1.4)
```

**Exam Scenario:** On-premises application can't access storage account with private endpoint. Solution: Verify VPN/ExpressRoute connectivity, check DNS resolution returns private IP, ensure on-premises DNS forwards to Azure DNS.

### Disabling Public Access

After creating private endpoint, disable public access for maximum security:

1. Navigate to Storage Account → **Security + networking** → **Networking**
2. Under **Public network access**, select **Disabled**
3. Click **Save**

**Result:** Storage account is ONLY accessible via private endpoints in your VNets.

**Exam Warning:** Disabling public access breaks:
- Azure Portal storage explorer (unless accessed from VM in VNet)
- Public internet access
- Azure services not using private endpoint

---

## Encryption at Rest

Azure Storage automatically encrypts all data at rest using 256-bit AES encryption.

### Encryption Scope

| Scope | Description |
|-------|-------------|
| **Storage Service Encryption (SSE)** | Encrypts data as it's written, decrypts as it's read |
| **All services** | Blob, File, Queue, Table, Managed Disks |
| **Always enabled** | Cannot be disabled |
| **Transparent** | No performance impact, no code changes required |

### Encryption Key Management

#### 1. Microsoft-Managed Keys (Default)
- **Management:** Microsoft manages all key operations
- **Key rotation:** Automatic rotation by Microsoft
- **User control:** None
- **Use case:** Simplest option, meets most compliance requirements

#### 2. Customer-Managed Keys (CMK)
- **Management:** Customer creates and manages keys in Azure Key Vault
- **Key rotation:** Manual or automated via Key Vault
- **User control:** Full control over key lifecycle
- **Use case:** Regulatory requirement for customer key management

#### 3. Customer-Provided Keys (CPK)
- **Management:** Customer provides key with each request
- **Key storage:** Customer responsibility (not in Azure)
- **User control:** Complete control, but operational burden
- **Use case:** Highest security requirement, bring your own key (BYOK)

**Exam Tip:** CMK is most common for enhanced control. CPK is rarely used due to complexity.

### Configuring Customer-Managed Keys

**Prerequisites:**
- Azure Key Vault in same region as storage account
- Key Vault must have:
  - Soft delete enabled
  - Purge protection enabled (recommended)
- Managed identity for storage account
- Key Vault access policy granting storage account identity:
  - Get, Wrap Key, Unwrap Key permissions

**Portal Steps:**
1. Navigate to Storage Account → **Security + networking** → **Encryption**
2. Under **Encryption type**, select **Customer-managed keys**
3. Under **Encryption key**, select:
   - **Enter key URI:** Specify Key Vault key URI
   - Or **Select from Key Vault:** Browse for key
4. Under **Identity**, select managed identity (create if needed)
5. Click **Save**

**Key Rotation:**
- **Automatic:** Enable key auto-rotation in Key Vault (storage account uses latest version)
- **Manual:** Update storage account to reference new key version

**Exam Scenario:** Compliance requires customer control over encryption keys. Solution: Create Key Vault, enable soft delete and purge protection, create key, assign managed identity to storage account, configure CMK.

### Infrastructure Encryption (Double Encryption)

For maximum security, enable infrastructure encryption for double encryption:

1. **Service-level encryption** (storage service encryption)
2. **Infrastructure-level encryption** (hardware-level encryption)

**Important:** Must be enabled at storage account creation time, cannot be enabled later.

**Portal Steps (during creation):**
1. When creating storage account
2. On **Encryption** tab
3. Check **Enable infrastructure encryption**

**Result:** Data encrypted twice with different keys at different layers.

---

## Encryption in Transit

Azure Storage enforces encryption in transit using TLS (Transport Layer Security).

### TLS Requirements

| Version | Status |
|---------|--------|
| **TLS 1.2** | Required (default minimum) |
| **TLS 1.1** | Deprecated, should be disabled |
| **TLS 1.0** | Deprecated, should be disabled |

### Enforcing Secure Transfer

**Portal Steps:**
1. Navigate to Storage Account → **Configuration**
2. Under **Secure transfer required**, select **Enabled**
3. Click **Save**

**Effect:**
- ✅ Requires HTTPS for REST API access
- ✅ Enforces SMB 3.0 with encryption for Azure Files
- ❌ Rejects HTTP requests
- ❌ Rejects SMB 2.1 connections (no encryption)

**Exam Tip:** Secure transfer required should always be enabled for production. Default is Enabled for new accounts.

### Setting Minimum TLS Version

**Portal Steps:**
1. Navigate to Storage Account → **Configuration**
2. Under **Minimum TLS version**, select **Version 1.2**
3. Click **Save**

**Impact:** Rejects connections using TLS 1.1 or TLS 1.0.

---

## Advanced Threat Protection

Microsoft Defender for Storage (formerly Advanced Threat Protection) detects anomalous activities and potential threats.

### Threat Detection Capabilities

| Detection | Description |
|-----------|-------------|
| **Unusual access patterns** | Access from unusual locations or anonymous clients |
| **Unusual data extraction** | Large volume extraction or unusual number of operations |
| **Malicious uploads** | Potential malware uploads (hash reputation analysis) |
| **Anomalous permissions changes** | Unusual changes to access control |
| **Unusual delete operations** | Mass deletion of blobs |

### Enabling Microsoft Defender for Storage

**Portal Steps:**
1. Navigate to **Microsoft Defender for Cloud**
2. Go to **Environment settings**
3. Select the subscription
4. Find **Storage accounts** under resource types
5. Toggle to **On**
6. Click **Save**

**Or per storage account:**
1. Navigate to Storage Account → **Security + networking** → **Microsoft Defender for Cloud**
2. Click **Enable**

**Cost:** Per storage account per month + per transaction fees.

### Security Alerts

When threat detected, alerts are sent to:
- Microsoft Defender for Cloud dashboard
- Azure Monitor
- Email to subscription owners/contributors

**Example Alert:** *"Access from a Tor exit node"* - Someone accessed storage via Tor anonymization network.

**Response Actions:**
1. Review alert details in Defender for Cloud
2. Investigate source IP and operations performed
3. Review Storage Analytics logs for full activity
4. Take remediation action (revoke SAS, rotate keys, add firewall rules)

---

## PowerShell & CLI Commands

### Access Key Management

#### PowerShell

```powershell
# Get storage account keys
$keys = Get-AzStorageAccountKey `
  -ResourceGroupName 'rg-storage' `
  -Name 'storageaccount01'

$key1 = $keys[0].Value
$key2 = $keys[1].Value

# Regenerate key1
New-AzStorageAccountKey `
  -ResourceGroupName 'rg-storage' `
  -Name 'storageaccount01' `
  -KeyName key1

# Create storage context with key
$context = New-AzStorageContext `
  -StorageAccountName 'storageaccount01' `
  -StorageAccountKey $key1

# Disable shared key access
Set-AzStorageAccount `
  -ResourceGroupName 'rg-storage' `
  -Name 'storageaccount01' `
  -AllowSharedKeyAccess $false
```

#### Azure CLI

```bash
# Get storage account keys
az storage account keys list \
  --resource-group rg-storage \
  --account-name storageaccount01 \
  --output table

# Regenerate key1
az storage account keys renew \
  --resource-group rg-storage \
  --account-name storageaccount01 \
  --key primary

# Disable shared key access
az storage account update \
  --resource-group rg-storage \
  --name storageaccount01 \
  --allow-shared-key-access false
```

### SAS Token Generation

#### PowerShell

```powershell
# User Delegation SAS (requires Azure AD authentication)
$ctx = New-AzStorageContext `
  -StorageAccountName 'storageaccount01' `
  -UseConnectedAccount

$sas = New-AzStorageBlobSASToken `
  -Container 'documents' `
  -Blob 'report.pdf' `
  -Permission 'r' `
  -ExpiryTime (Get-Date).AddHours(8) `
  -Protocol HttpsOnly `
  -Context $ctx

# Service SAS for container
$sas = New-AzStorageContainerSASToken `
  -Name 'documents' `
  -Permission 'rl' `
  -ExpiryTime (Get-Date).AddDays(1) `
  -Protocol HttpsOnly `
  -Context $context

# Account SAS
$sas = New-AzStorageAccountSASToken `
  -Service Blob,File `
  -ResourceType Service,Container,Object `
  -Permission 'rl' `
  -ExpiryTime (Get-Date).AddDays(7) `
  -Protocol HttpsOnly `
  -Context $context
```

#### Azure CLI

```bash
# User Delegation SAS
az storage blob generate-sas \
  --account-name storageaccount01 \
  --container-name documents \
  --name report.pdf \
  --permissions r \
  --expiry 2024-12-31T23:59:59Z \
  --https-only \
  --as-user \
  --auth-mode login \
  --full-uri

# Service SAS for container
az storage container generate-sas \
  --account-name storageaccount01 \
  --name documents \
  --permissions rl \
  --expiry 2024-12-31T23:59:59Z \
  --https-only \
  --account-key "<key>"

# Account SAS
az storage account generate-sas \
  --account-name storageaccount01 \
  --services bf \
  --resource-types sco \
  --permissions rl \
  --expiry 2024-12-31T23:59:59Z \
  --https-only \
  --account-key "<key>"
```

### Network Rules

#### PowerShell

```powershell
# Allow specific IP range
Add-AzStorageAccountNetworkRule \
  -ResourceGroupName 'rg-storage' \
  -Name 'storageaccount01' \
  -IPAddressOrRange '203.0.113.0/24'

# Allow VNet subnet
$subnet = Get-AzVirtualNetworkSubnetConfig \
  -VirtualNetworkName 'vnet-prod' \
  -Name 'subnet-web' \
  -ResourceGroupName 'rg-network'

Add-AzStorageAccountNetworkRule \
  -ResourceGroupName 'rg-storage' \
  -Name 'storageaccount01' \
  -VirtualNetworkResourceId $subnet.Id

# Set default action to deny
Update-AzStorageAccountNetworkRuleSet \
  -ResourceGroupName 'rg-storage' \
  -Name 'storageaccount01' \
  -DefaultAction Deny

# Allow Azure services
Update-AzStorageAccountNetworkRuleSet \
  -ResourceGroupName 'rg-storage' \
  -Name 'storageaccount01' \
  -Bypass AzureServices

# Remove IP rule
Remove-AzStorageAccountNetworkRule \
  -ResourceGroupName 'rg-storage' \
  -Name 'storageaccount01' \
  -IPAddressOrRange '203.0.113.0/24'
```

#### Azure CLI

```bash
# Allow specific IP range
az storage account network-rule add \
  --resource-group rg-storage \
  --account-name storageaccount01 \
  --ip-address 203.0.113.0/24

# Allow VNet subnet
az storage account network-rule add \
  --resource-group rg-storage \
  --account-name storageaccount01 \
  --vnet-name vnet-prod \
  --subnet subnet-web

# Set default action to deny
az storage account update \
  --resource-group rg-storage \
  --name storageaccount01 \
  --default-action Deny

# Allow Azure services
az storage account update \
  --resource-group rg-storage \
  --name storageaccount01 \
  --bypass AzureServices

# Remove IP rule
az storage account network-rule remove \
  --resource-group rg-storage \
  --account-name storageaccount01 \
  --ip-address 203.0.113.0/24
```

### Private Endpoints

#### PowerShell

```powershell
# Create private endpoint
$vnet = Get-AzVirtualNetwork `
  -Name 'vnet-prod' `
  -ResourceGroupName 'rg-network'

$subnet = Get-AzVirtualNetworkSubnetConfig `
  -VirtualNetwork $vnet `
  -Name 'subnet-privateendpoints'

$storageAccount = Get-AzStorageAccount `
  -ResourceGroupName 'rg-storage' `
  -Name 'storageaccount01'

$privateEndpoint = New-AzPrivateEndpoint `
  -Name 'pe-storageaccount01-blob' `
  -ResourceGroupName 'rg-network' `
  -Location 'eastus' `
  -Subnet $subnet `
  -PrivateLinkServiceConnection (
    New-AzPrivateLinkServiceConnection `
      -Name 'connection-blob' `
      -PrivateLinkServiceId $storageAccount.Id `
      -GroupId 'blob'
  )

# Create Private DNS Zone
$zone = New-AzPrivateDnsZone `
  -Name 'privatelink.blob.core.windows.net' `
  -ResourceGroupName 'rg-network'

# Link to VNet
New-AzPrivateDnsVirtualNetworkLink `
  -Name 'link-vnet-prod' `
  -ResourceGroupName 'rg-network' `
  -ZoneName 'privatelink.blob.core.windows.net' `
  -VirtualNetworkId $vnet.Id

# Create DNS zone group (auto-creates A record)
$config = New-AzPrivateDnsZoneConfig `
  -Name 'privatelink.blob.core.windows.net' `
  -PrivateDnsZoneId $zone.ResourceId

New-AzPrivateDnsZoneGroup `
  -Name 'zonegroup-blob' `
  -ResourceGroupName 'rg-network' `
  -PrivateEndpointName 'pe-storageaccount01-blob' `
  -PrivateDnsZoneConfig $config
```

#### Azure CLI

```bash
# Create private endpoint
az network private-endpoint create \
  --name pe-storageaccount01-blob \
  --resource-group rg-network \
  --vnet-name vnet-prod \
  --subnet subnet-privateendpoints \
  --private-connection-resource-id $(az storage account show -n storageaccount01 -g rg-storage --query id -o tsv) \
  --group-id blob \
  --connection-name connection-blob

# Create Private DNS Zone
az network private-dns zone create \
  --resource-group rg-network \
  --name privatelink.blob.core.windows.net

# Link to VNet
az network private-dns link vnet create \
  --resource-group rg-network \
  --zone-name privatelink.blob.core.windows.net \
  --name link-vnet-prod \
  --virtual-network vnet-prod \
  --registration-enabled false

# Create DNS zone group
az network private-endpoint dns-zone-group create \
  --resource-group rg-network \
  --endpoint-name pe-storageaccount01-blob \
  --name zonegroup-blob \
  --private-dns-zone privatelink.blob.core.windows.net \
  --zone-name privatelink.blob.core.windows.net
```

### Customer-Managed Keys

#### PowerShell

```powershell
# Enable system-assigned managed identity
Set-AzStorageAccount \
  -ResourceGroupName 'rg-storage' \
  -Name 'storageaccount01' \
  -AssignIdentity

# Get storage account identity
$sa = Get-AzStorageAccount `
  -ResourceGroupName 'rg-storage' `
  -Name 'storageaccount01'
$identityId = $sa.Identity.PrincipalId

# Grant Key Vault access to storage account identity
Set-AzKeyVaultAccessPolicy `
  -VaultName 'keyvault01' `
  -ObjectId $identityId `
  -PermissionsToKeys get,wrapKey,unwrapKey

# Configure CMK
$keyVault = Get-AzKeyVault -VaultName 'keyvault01'
$key = Get-AzKeyVaultKey -VaultName 'keyvault01' -KeyName 'storagekey'

Set-AzStorageAccount `
  -ResourceGroupName 'rg-storage' `
  -Name 'storageaccount01' `
  -KeyvaultEncryption `
  -KeyName $key.Name `
  -KeyVersion $key.Version `
  -KeyVaultUri $keyVault.VaultUri
```

#### Azure CLI

```bash
# Enable system-assigned managed identity
az storage account update \
  --resource-group rg-storage \
  --name storageaccount01 \
  --assign-identity

# Get storage account identity
IDENTITY=$(az storage account show \
  --resource-group rg-storage \
  --name storageaccount01 \
  --query identity.principalId -o tsv)

# Grant Key Vault access
az keyvault set-policy \
  --name keyvault01 \
  --object-id $IDENTITY \
  --key-permissions get wrapKey unwrapKey

# Configure CMK
az storage account update \
  --resource-group rg-storage \
  --name storageaccount01 \
  --encryption-key-source Microsoft.Keyvault \
  --encryption-key-vault https://keyvault01.vault.azure.net \
  --encryption-key-name storagekey
```

---

## Exam Tips

### Key Concepts for AZ-104

1. **Access Key vs. SAS:**
   - **Access keys:** Full account access, no expiration, high risk
   - **SAS:** Granular permissions, time-limited, revocable (with stored access policy)
   - **Exam Scenario:** Grant temporary read-only access to specific container → Use SAS, not access key

2. **SAS Types:**
   - **User Delegation SAS:** Blob only, Azure AD secured, most secure
   - **Service SAS:** Single service, can use stored access policy for revocation
   - **Account SAS:** Multiple services, no stored access policy support
   - **Exam Tip:** Always prefer User Delegation SAS for blobs

3. **Network Security:**
   - **Service Endpoints:** Free, traffic still uses public IPs
   - **Private Endpoints:** Cost, true private IPs, best for on-prem connectivity
   - **Exam Scenario:** On-premises needs private access to storage → Use Private Endpoint

4. **Encryption at Rest:**
   - **Always enabled**, cannot be disabled
   - **Microsoft-managed keys:** Default, automatic rotation
   - **Customer-managed keys:** Customer controls rotation, requires Key Vault
   - **Exam Tip:** CMK requires managed identity and Key Vault access policy

5. **Encryption in Transit:**
   - **Secure transfer required:** Enforces HTTPS and SMB 3.0+
   - **Minimum TLS version:** Should be 1.2
   - **Exam Scenario:** Legacy app using HTTP fails to connect → Check "Secure transfer required" setting

6. **Azure AD Authentication:**
   - **Blob and Queue:** Fully supported
   - **Table:** Limited support
   - **File:** Requires domain services (AD DS or Azure AD DS)
   - **Exam Tip:** Use managed identities for Azure services accessing storage

7. **Firewall Rule Processing:**
   - **Default action:** Allow (public) or Deny (selected networks)
   - **Exceptions:** Evaluated before default action
   - **Azure services bypass:** Requires trusted services exception
   - **Exam Scenario:** Azure Backup can't access storage after enabling firewall → Enable "Allow Azure services" exception

8. **Shared Key Access:**
   - **Disabling:** Prevents all access key authentication
   - **Impact:** Requires Azure AD or SAS for all access
   - **Exam Tip:** Plan migration before disabling shared key access

### Common Exam Scenarios

**Scenario 1: External User Needs Temporary Blob Access**
- **Question:** External partner needs to download specific files for 48 hours
- **Answer:** Generate User Delegation SAS with read permission, 48-hour expiry, and partner's IP address
- **Why:** SAS is time-limited and granular, User Delegation is most secure

**Scenario 2: Application Authentication**
- **Question:** Azure Function needs to read blobs, what authentication should it use?
- **Answer:** Enable managed identity, assign Storage Blob Data Reader role, use DefaultAzureCredential
- **Why:** Managed identities eliminate credential storage and management

**Scenario 3: Storage Account Access Blocked**
- **Question:** After enabling firewall, Azure Portal can't access storage account
- **Answer:** Add your client IP address to firewall allow list, or access from VM inside allowed VNet
- **Why:** Portal access is from your public IP, which is now blocked by firewall

**Scenario 4: Revoking SAS Access**
- **Question:** SAS token was shared with wrong person, how to revoke immediately?
- **Answer:** If service SAS with stored access policy: modify policy expiry. If account SAS or user delegation SAS without policy: regenerate storage account keys
- **Why:** Only stored access policies support revocation without key regeneration

**Scenario 5: On-Premises Access to Storage**
- **Question:** On-premises application needs private connectivity to Azure Storage
- **Answer:** Create private endpoint in Azure VNet, configure VPN or ExpressRoute, set up DNS forwarding
- **Why:** Private endpoint provides private IP accessible from on-premises via VPN/ExpressRoute

**Scenario 6: Compliance Requires Customer-Managed Keys**
- **Question:** Regulatory requirement to control encryption keys
- **Answer:** 
  1. Create Azure Key Vault with soft delete and purge protection
  2. Create or import encryption key
  3. Enable managed identity on storage account
  4. Grant Key Vault access to storage account identity (get, wrapKey, unwrapKey)
  5. Configure storage account to use customer-managed key
- **Why:** CMK provides customer control over encryption key lifecycle

---

## Common Scenarios & Troubleshooting

### Scenario: Can't Access Storage After Enabling Firewall

**Symptoms:**
- 403 Forbidden errors
- "This request is not authorized" messages
- Azure Portal storage explorer fails

**Troubleshooting Steps:**
1. **Check network rules:** Navigate to Storage Account → Networking → Firewalls and virtual networks
2. **Verify your IP:** Check if your client IP is in allowed list
3. **Azure services:** Verify "Allow Azure services" is enabled if needed
4. **VNet rules:** Confirm subnet has service endpoint enabled
5. **DNS cache:** Clear DNS cache if recently added private endpoint

**Resolution:**
```powershell
# Add your current public IP
$myIp = (Invoke-WebRequest -Uri 'https://api.ipify.org').Content
Add-AzStorageAccountNetworkRule `
  -ResourceGroupName 'rg-storage' `
  -Name 'storageaccount01' `
  -IPAddressOrRange $myIp
```

### Scenario: SAS Token Returns 403 Forbidden

**Symptoms:**
- SAS URL returns "Server failed to authenticate the request"
- Error code: AuthenticationFailed

**Troubleshooting Steps:**
1. **Check expiry:** Verify SAS hasn't expired
2. **Verify permissions:** Ensure SAS has required permissions (r, w, d, l)
3. **Clock skew:** Check system time is accurate (allow 15-minute buffer)
4. **IP restriction:** Verify request comes from allowed IP
5. **Protocol:** Check HTTPS is used if SAS specifies HTTPS only
6. **Key rotation:** If service/account SAS, verify account key wasn't regenerated

**Resolution:**
```powershell
# Generate new SAS token
$ctx = New-AzStorageContext `
  -StorageAccountName 'storageaccount01' `
  -StorageAccountKey $key1

$newSas = New-AzStorageBlobSASToken `
  -Container 'documents' `
  -Blob 'report.pdf' `
  -Permission 'r' `
  -ExpiryTime (Get-Date).AddHours(24) `
  -Protocol HttpsOnly `
  -Context $ctx

Write-Output $newSas
```

### Scenario: Private Endpoint Not Resolving

**Symptoms:**
- Storage account FQDN resolves to public IP instead of private IP
- Connection fails from VNet

**Troubleshooting Steps:**
1. **DNS resolution:** Test from VM in VNet: `nslookup storageaccount.blob.core.windows.net`
2. **Private DNS zone:** Verify zone is created and linked to VNet
3. **A record:** Check A record exists in private DNS zone
4. **Network connectivity:** Verify VM can reach private endpoint IP
5. **NSG rules:** Check NSG allows outbound to storage on port 443

**Resolution:**
```powershell
# Verify private endpoint DNS
$pe = Get-AzPrivateEndpoint -Name 'pe-storageaccount01-blob' -ResourceGroupName 'rg-network'
$pe.NetworkInterfaces[0].IpConfigurations[0].PrivateIpAddress

# Check private DNS zone link
Get-AzPrivateDnsVirtualNetworkLink `
  -ResourceGroupName 'rg-network' `
  -ZoneName 'privatelink.blob.core.windows.net'

# Test connectivity from VM
Test-NetConnection -ComputerName storageaccount.blob.core.windows.net -Port 443
```

### Scenario: Managed Identity Can't Access Storage

**Symptoms:**
- Application using managed identity gets "This request is not authorized"
- 403 Forbidden errors

**Troubleshooting Steps:**
1. **Identity enabled:** Verify managed identity is enabled on resource (VM, App Service, etc.)
2. **RBAC assignment:** Check identity has appropriate role (Storage Blob Data Contributor/Reader)
3. **Scope:** Verify role assigned at correct scope (resource, container, storage account)
4. **Propagation time:** Wait 5-10 minutes for role assignment to propagate
5. **Code:** Ensure application uses DefaultAzureCredential or equivalent

**Resolution:**
```powershell
# Get managed identity principal ID
$vm = Get-AzVM -ResourceGroupName 'rg-compute' -Name 'vm-app-01'
$identityId = $vm.Identity.PrincipalId

# Verify role assignment
Get-AzRoleAssignment -ObjectId $identityId

# Assign role if missing
$sa = Get-AzStorageAccount -ResourceGroupName 'rg-storage' -Name 'storageaccount01'
New-AzRoleAssignment `
  -ObjectId $identityId `
  -RoleDefinitionName 'Storage Blob Data Contributor' `
  -Scope $sa.Id
```

### Scenario: Customer-Managed Key Access Denied

**Symptoms:**
- Storage operations fail with "Key Vault access denied"
- Encryption shows "Key inaccessible"

**Troubleshooting Steps:**
1. **Key Vault permissions:** Verify storage account managed identity has Key Vault access
2. **Key state:** Check key is enabled, not disabled or expired
3. **Key Vault firewall:** Verify storage account can reach Key Vault
4. **Soft delete:** Check key wasn't soft-deleted
5. **Purge protection:** Verify Key Vault has purge protection enabled

**Resolution:**
```powershell
# Verify storage account identity
$sa = Get-AzStorageAccount -ResourceGroupName 'rg-storage' -Name 'storageaccount01'
$identityId = $sa.Identity.PrincipalId

# Check Key Vault access policy
Get-AzKeyVaultAccessPolicy -VaultName 'keyvault01' | Where-Object {$_.ObjectId -eq $identityId}

# Grant access if missing
Set-AzKeyVaultAccessPolicy `
  -VaultName 'keyvault01' `
  -ObjectId $identityId `
  -PermissionsToKeys get,wrapKey,unwrapKey
```

---

## Related Topics

### Linked Study Materials

- **[Manage Storage](./Manage%20Storage.md):** Storage redundancy, AzCopy, replication
- **[Configure Azure Files and Blob Storage](./Configure%20Azure%20Files%20and%20Blob%20Storage.md):** Blob tiers, lifecycle management, file shares
- **[Manage RBAC](../Manage%20Azure%20Identities%20and%20Governance/Manage%20RBAC.md):** Role assignments for storage access
- **[Secure Access to Virtual Networks](../Configure%20and%20Manage%20Virtual%20Networking/Secure%20Access%20to%20Virtual%20Networks.md):** NSGs, service endpoints

### Azure Services Integration

- **Azure Key Vault:** Customer-managed key storage
- **Azure AD:** Authentication and authorization
- **Microsoft Defender for Storage:** Threat detection
- **Azure Monitor:** Storage analytics and logging
- **Azure Policy:** Enforce security configurations

### Advanced Topics (Beyond AZ-104)

- **Azure Storage encryption scopes:** Encrypt different containers with different keys
- **Object replication:** Replicate blobs between storage accounts
- **Azure Confidential Ledger:** Immutable storage for compliance
- **Immutable blob storage:** Write-once-read-many (WORM) for legal/compliance holds

### Documentation Links

- [Azure Storage Security Guide](https://learn.microsoft.com/en-us/azure/storage/common/security-recommendations)
- [SAS Overview](https://learn.microsoft.com/en-us/azure/storage/common/storage-sas-overview)
- [Customer-Managed Keys](https://learn.microsoft.com/en-us/azure/storage/common/customer-managed-keys-overview)
- [Private Endpoints for Storage](https://learn.microsoft.com/en-us/azure/storage/common/storage-private-endpoints)
- [Microsoft Defender for Storage](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-storage-introduction)

---

**Study Checkpoint:** You should now understand storage account keys, SAS tokens, Azure AD authentication, network security, encryption, and threat protection. Practice generating SAS tokens, configuring firewalls, and creating private endpoints in the Azure portal.

**Next Steps:** Review [Manage Storage](./Manage%20Storage.md) for redundancy options and [Configure Azure Files and Blob Storage](./Configure%20Azure%20Files%20and%20Blob%20Storage.md) for service-specific configurations.
