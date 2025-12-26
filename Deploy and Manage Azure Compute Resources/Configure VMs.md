# Configure VMs - Azure Administrator Exam Prep (AZ-104)

## Table of Contents
1. [VM Sizing and Selection](#vm-sizing-and-selection)
2. [Managed Disks](#managed-disks)
3. [Disk Encryption](#disk-encryption)
4. [VM Availability](#vm-availability)
5. [Virtual Machine Scale Sets](#virtual-machine-scale-sets)
6. [PowerShell & CLI Commands](#powershell--cli-commands)
7. [Exam Tips & Key Concepts](#exam-tips--key-concepts)
8. [Exam Scenarios](#exam-scenarios)
9. [Troubleshooting Guide](#troubleshooting-guide)

---

## VM Sizing and Selection

### Understanding VM Size Categories

Azure VMs are categorized by workload type to help you select the appropriate size:

**General Purpose (B, D, E families)**
- Balanced compute, memory, and networking
- Web servers, small-to-medium databases, dev/test environments
- Examples: Standard_B2s, Standard_D2s_v5, Standard_E2s_v5

**Compute Optimized (F, H families)**
- High CPU-to-memory ratio
- Medium traffic web servers, batch processing, application servers
- Examples: Standard_F2s_v2, Standard_H16r

**Memory Optimized (E, M families)**
- High memory-to-CPU ratio
- Relational databases, in-memory analytics, large caches
- Examples: Standard_E4s_v5, Standard_M128s

**Storage Optimized (L, Lsv2 families)**
- High disk throughput and I/O
- NoSQL databases, data warehousing, large-scale data analytics
- Examples: Standard_L8s_v2 (up to 10x1.92 TiB NVMe SSD)

**GPU/Accelerated (N, ND families)**
- Graphics processing or specialized compute
- Machine learning, deep learning, high-performance computing
- Examples: Standard_NC6s_v3, Standard_ND40s_v2

### VM Naming Convention

Azure uses a structured naming scheme:

```
Standard_D4s_v5
├─ Standard: Family (General Purpose)
├─ D: Series (compute/memory optimized)
├─ 4: Number of vCPUs
├─ s: Supports Premium Storage
└─ v5: Generation
```

**Key naming indicators:**
- **s suffix**: Premium Storage capable
- **a suffix**: AMD processors (instead of Intel)
- **p suffix**: ARM-based processors (Cobalt, Ampere Altra)

### Selecting the Right VM Size

**Analyze performance signals:**
1. **CPU-intensive workloads**: Choose compute-optimized (F-family, high CPU-to-memory ratio)
2. **Memory-intensive workloads**: Choose memory-optimized (E, M families, high memory-to-CPU)
3. **Storage-bound workloads**: Choose storage-optimized (L, Lsv2 with NVMe)
4. **Premium Storage needs**: Select SKU with 's' in name (e.g., Standard_E4**s**_v3)

**Tools for selection:**
- Azure VM Selector (azure.microsoft.com/pricing/vm-selector/)
- Sizing for performance (compare vCPU, RAM, disk IOPS, throughput)

### Vertical vs. Horizontal Scaling

**Vertical Scaling (Scale-up):**
- Increase VM size (more CPU, memory)
- Requires VM downtime for resize
- Limited by VM size availability in region
- No redundancy

**Horizontal Scaling (Scale-out):**
- Deploy multiple smaller VMs
- Use load balancer for traffic distribution
- No downtime required
- Provides high availability and redundancy
- Used with Virtual Machine Scale Sets

---

## Managed Disks

### Disk Types Overview

Azure offers five managed disk types for different workload requirements:

| Disk Type | Use Case | Max IOPS | Max Throughput | Max Size | OS Disk? |
|-----------|----------|----------|----------------|----------|----------|
| **Ultra Disk** | IO-intensive (SAP HANA, top-tier databases) | 400,000 | 10,000 MB/s | 65,536 GiB | No |
| **Premium SSD v2** | Production, performance-sensitive | 80,000 | 1,200 MB/s | 65,536 GiB | No |
| **Premium SSD** | Production workloads | 20,000 | 900 MB/s | 32,767 GiB | Yes |
| **Standard SSD** | Web servers, dev/test | 6,000 | 750 MB/s | 32,767 GiB | Yes |
| **Standard HDD** | Backup, infrequent access | 2,000 | 500 MB/s | 32,767 GiB | Yes* |

*Standard HDD as OS disk retiring September 8, 2028

### Premium SSD Sizing

Premium SSDs scale performance with size using P-tier labels:

| Size | IOPS | Throughput | Cost |
|------|------|-----------|------|
| P1 (4 GiB) | 120 | 25 MB/s | Low |
| P4 (32 GiB) | 120 | 25 MB/s | Low |
| P10 (128 GiB) | 500 | 100 MB/s | Medium |
| P30 (1 TiB) | 5,000 | 200 MB/s | High |
| P80 (32 TiB) | 20,000 | 900 MB/s | Highest |

### Standard SSD Sizing

Standard SSDs use E-tier labels with base IOPS up to 500:

| Size | Base IOPS | Expanded IOPS | Cost |
|------|-----------|---------------|------|
| E1-E50 (4-4,096 GiB) | Up to 500 | N/A | Low |
| E60+ | Up to 2,000 | Up to 6,000 | Medium |

### Disk Management Operations

**Create a managed disk:**

```powershell
# Create empty managed disk
$diskConfig = New-AzDiskConfig `
    -Location "EastUS" `
    -DiskSizeGB 128 `
    -StorageAccountType "Premium_LRS"

$disk = New-AzDisk `
    -ResourceGroupName "myResourceGroup" `
    -DiskName "myDataDisk" `
    -Disk $diskConfig
```

```azurecli
# Create from snapshot
az disk create \
    --resource-group myResourceGroup \
    --name myDataDisk \
    --source mySnapshot
```

**Attach disk to VM:**

```powershell
$vm = Get-AzVM -ResourceGroupName "myResourceGroup" -Name "myVM"

Add-AzVMDataDisk -VM $vm `
    -Name "myDataDisk" `
    -CreateOption Attach `
    -ManagedDiskId $disk.Id `
    -Lun 0

Update-AzVM -ResourceGroupName "myResourceGroup" -VM $vm
```

**Convert disk type:**

```azurecli
# Switch all disks from Premium to Standard SSD
az disk update \
    --resource-group myResourceGroup \
    --name myDataDisk \
    --sku StandardSSD_LRS

# Deallocate VM first for production disks
az vm deallocate --resource-group myResourceGroup --name myVM
```

### Disk Fault Domains

- Only managed disks can use managed availability sets
- Disk fault domains vary by region (2 or 3 FDs per region)
- Managed disks align with VM fault domains for redundancy
- Check FD count: `az vm list-skus --resource-type availabilitySets --query '[?name==Aligned].{Location:locationInfo[0].location, MaxFD:capabilities[0].value}'`

---

## Disk Encryption

### Encryption Options

Azure provides multiple disk encryption methods:

**1. Server-Side Encryption (SSE)**
- **Platform-Managed Keys (default)**: Microsoft manages keys, no action required
- **Customer-Managed Keys (CMK)**: You manage keys in Azure Key Vault
- Always enabled, transparent to users
- No performance impact

**2. Azure Disk Encryption (ADE)**
- **Status**: Retiring September 15, 2028
- Uses BitLocker (Windows) or DM-Crypt (Linux)
- Requires Azure Key Vault for key management
- Can encrypt both OS and data disks
- Migration required to Encryption at Host

**3. Encryption at Host**
- End-to-end encryption from VM to storage
- Encrypts temp disks (D: drive)
- Recommended for new VMs replacing ADE
- Automatic, no key management required
- Supports customer-managed keys

**4. Confidential VM Encryption**
- Encrypts VM guest state
- Hardware-based attestation with vTPM
- For highly sensitive workloads

### Enable Encryption at Host

**Using PowerShell:**

```powershell
# Create disk encryption set
$rgName = "myResourceGroup"
$keyVaultName = "myKeyVault"
$desName = "myDiskEncryptionSet"

# Get key from vault
$keyVaultKeyUrl = (Get-AzKeyVaultKey -VaultName $keyVaultName -Name "myKey").Key.Kid

# Create encryption set
$des = New-AzDiskEncryptionSet `
    -ResourceGroupName $rgName `
    -Name $desName `
    -KeyUrl $keyVaultKeyUrl `
    -SourceVaultId $(Get-AzKeyVault -VaultName $keyVaultName).ResourceId

# Create VM with encryption at host
$vm = New-AzVMConfig -VMName "myVM" -VMSize "Standard_DS2_v2" -EncryptionAtHost

$vm = Set-AzVMOSDisk `
    -VM $vm `
    -Name "myOSDisk" `
    -DiskEncryptionSetId $des.Id `
    -CreateOption FromImage

New-AzVM -ResourceGroupName $rgName -VM $vm
```

**Using Azure CLI:**

```bash
# Create encryption set with CMK
diskEncryptionSetId=$(az disk-encryption-set show \
    -n myDiskEncryptionSet \
    -g myResourceGroup \
    --query [id] -o tsv)

# Create VM with encryption at host
az vm create \
    --resource-group myResourceGroup \
    --name myVM \
    --size Standard_DS2_v2 \
    --encryption-at-host \
    --os-disk-encryption-set $diskEncryptionSetId \
    --image "UbuntuLTS"
```

### Azure Disk Encryption (Legacy - Retiring)

**Enable ADE on existing VM (before retirement):**

```powershell
# Prerequisites: Key Vault must exist and be enabled for disk encryption
$vmName = "myVM"
$rgName = "myResourceGroup"
$keyVaultName = "myKeyVault"

Enable-AzVMDiskEncryption `
    -ResourceGroupName $rgName `
    -VMName $vmName `
    -DiskEncryptionKeyVaultUrl "https://$keyVaultName.vault.azure.net/" `
    -DiskEncryptionKeyVaultId $(Get-AzKeyVault -VaultName $keyVaultName).ResourceId `
    -VolumeType "All"
```

**Disable ADE:**

```powershell
# Disable encryption
Disable-AzVMDiskEncryption `
    -ResourceGroupName "myResourceGroup" `
    -VMName "myVM" `
    -VolumeType "All"

# Remove extension after disabling
Remove-AzVMExtension `
    -ResourceGroupName "myResourceGroup" `
    -VMName "myVM" `
    -Name "AzureDiskEncryption"
```

### Encryption Restrictions

**Cannot encrypt:**
- L-series VMs (storage optimized)
- M-series VMs with Write Accelerator
- VMs with Encryption at Host + ADE together
- Ultra Disks or Premium SSD v2 (not supported with ADE)
- VMs in failover clusters

---

## VM Availability

### Availability Sets

Groups VMs across multiple **fault domains** and **update domains** to provide high availability.

**Fault Domains (FDs):**
- Physical isolation within datacenter
- Shared power, cooling, networking per FD
- Default: 3 FDs per availability set (2 in some regions)
- Protects against hardware failures
- VMs spread automatically across FDs

**Update Domains (UDs):**
- Logical grouping for maintenance
- Default: 20 UDs per availability set
- Only one UD rebooted during platform updates at a time
- 30-minute recovery window between UDs

**Create availability set:**

```powershell
New-AzAvailabilitySet `
    -ResourceGroupName "myResourceGroup" `
    -Name "myAvailabilitySet" `
    -Location "EastUS" `
    -PlatformFaultDomainCount 3 `
    -PlatformUpdateDomainCount 20
```

```azurecli
az vm availability-set create \
    --resource-group myResourceGroup \
    --name myAvailabilitySet \
    --location eastus \
    --platform-fault-domain-count 3 \
    --platform-update-domain-count 20
```

**Deploy VM in availability set:**

```powershell
$availabilitySet = Get-AzAvailabilitySet `
    -ResourceGroupName "myResourceGroup" `
    -Name "myAvailabilitySet"

$vmConfig = New-AzVMConfig `
    -VMName "myVM1" `
    -VMSize "Standard_D2s_v3" `
    -AvailabilitySetId $availabilitySet.Id

# Configure OS and NIC, then create
New-AzVM -ResourceGroupName "myResourceGroup" -VM $vmConfig
```

### Availability Zones

Physically separate datacenters within same region with independent power, cooling, and networking.

**Zone characteristics:**
- Minimum 3 zones per enabled region
- Each zone is a combination of FD + UD
- 99.99% VM uptime SLA when deployed across zones
- Low-latency networking between zones (typically <2ms)

**Deploy VM to specific zone:**

```powershell
New-AzVM `
    -ResourceGroupName "myResourceGroup" `
    -Name "myVM" `
    -Zone "1" `
    -Size "Standard_D2s_v3"
```

```azurecli
az vm create \
    --resource-group myResourceGroup \
    --name myVM \
    --zone 1 \
    --size Standard_D2s_v3
```

**Availability Set vs Availability Zone:**

| Feature | Availability Set | Availability Zone |
|---------|-----------------|-------------------|
| Scope | Single datacenter | Multiple datacenters |
| Fault Domain Isolation | Yes (2-3 FDs) | Yes (across zones) |
| SLA | 99.95% | 99.99% |
| Latency | <1ms | <2ms typical |
| Redundancy | Hardware level | Datacenter level |
| Cost | No extra charge | No extra charge |

---

## Virtual Machine Scale Sets

### Overview

A compute resource that deploys and manages identical VMs with automatic scaling and load balancing.

**Key benefits:**
- Easy management of hundreds/thousands of VMs
- Automatic scaling based on demand
- High availability across fault domains or availability zones
- Works at large scale (Flexible up to 1,000 VMs)

### Orchestration Modes

**Uniform Orchestration:**
- All VMs created from same template
- Automatic deployment across UDs (update domains)
- Maximum availability guarantee: 99.95% (FD>1) or 99.99% (across zones)
- Basic layer-4 LB support
- Scaling limits: traditional approach

**Flexible Orchestration:**
- VMs can have different sizes/images
- Spreads across fault domains
- 99.95% SLA for FD >1; 99.99% for zones
- Supports up to 1,000 instances
- Can mix Spot and On-Demand VMs
- Supports quorum-based workloads

**Availability Sets:**
- Traditional high availability
- 99.95% SLA
- Limited to single scale unit
- Max 20 UDs per set

### Create Virtual Machine Scale Set

**Using PowerShell:**

```powershell
$rgName = "myResourceGroup"
$vmssName = "myVMSS"
$location = "EastUS"

# Create VMSS
New-AzVmss `
    -ResourceGroupName $rgName `
    -VMScaleSetName $vmssName `
    -Location $location `
    -VMSize "Standard_D2s_v3" `
    -ImageName "UbuntuLTS" `
    -AdminUsername "azureuser" `
    -InstanceCount 3 `
    -LoadBalancerName "myLoadBalancer" `
    -LoadBalancerBackendPoolName "myBackendPool"
```

**Using Azure CLI:**

```bash
az vmss create \
    --resource-group myResourceGroup \
    --name myVMSS \
    --image UbuntuLTS \
    --vm-sku Standard_D2s_v3 \
    --instance-count 3 \
    --admin-username azureuser \
    --load-balancer myLoadBalancer
```

### Configure Autoscaling

**Create autoscale rule (scale-out on CPU):**

```powershell
# Get VMSS
$vmss = Get-AzVmss `
    -ResourceGroupName "myResourceGroup" `
    -VMScaleSetName "myVMSS"

# Create scale-out rule (CPU > 70%)
$scaleOutRule = New-AzAutoscaleRule `
    -MetricName "Percentage CPU" `
    -MetricResourceId $vmss.Id `
    -Operator GreaterThan `
    -MetricStatistic Average `
    -Threshold 70 `
    -TimeGrain ([System.TimeSpan]::FromMinutes(5)) `
    -TimeWindow ([System.TimeSpan]::FromMinutes(10)) `
    -ScaleActionCooldown ([System.TimeSpan]::FromMinutes(5)) `
    -ScaleActionDirection Increase `
    -ScaleActionValue 2

# Create scale-in rule (CPU < 30%)
$scaleInRule = New-AzAutoscaleRule `
    -MetricName "Percentage CPU" `
    -MetricResourceId $vmss.Id `
    -Operator LessThan `
    -MetricStatistic Average `
    -Threshold 30 `
    -TimeGrain ([System.TimeSpan]::FromMinutes(5)) `
    -TimeWindow ([System.TimeSpan]::FromMinutes(10)) `
    -ScaleActionCooldown ([System.TimeSpan]::FromMinutes(5)) `
    -ScaleActionDirection Decrease `
    -ScaleActionValue 1

# Create autoscale profile
$profile = New-AzAutoscaleProfile `
    -DefaultCapacity 3 `
    -MaximumCapacity 10 `
    -MinimumCapacity 2 `
    -Rule $scaleOutRule, $scaleInRule `
    -Name "CPU-based scaling"

# Create autoscale setting
New-AzAutoscaleSetting `
    -Location "EastUS" `
    -Name "myAutoscaleSetting" `
    -ResourceGroupName "myResourceGroup" `
    -TargetResourceId $vmss.Id `
    -AutoscaleProfile $profile
```

**Using Azure CLI:**

```bash
# Enable autoscale with CPU-based rules
az monitor autoscale create \
    --resource-group myResourceGroup \
    --resource myVMSS \
    --resource-type "Microsoft.Compute/virtualMachineScaleSets" \
    --name myAutoscaleSetting \
    --min-count 2 \
    --max-count 10 \
    --count 3

# Add scale-out rule (CPU > 70%)
az monitor autoscale rule create \
    --resource-group myResourceGroup \
    --autoscale-name myAutoscaleSetting \
    --condition "Percentage CPU > 70 avg 5m" \
    --scale out 2

# Add scale-in rule (CPU < 30%)
az monitor autoscale rule create \
    --resource-group myResourceGroup \
    --autoscale-name myAutoscaleSetting \
    --condition "Percentage CPU < 30 avg 5m" \
    --scale in 1
```

### Autoscale Metrics and Behavior

**Available metrics:**
- Percentage CPU (most common)
- Memory usage
- Network In/Out
- Disk Read/Write Bytes
- Active connections
- Custom metrics

**Autoscale behavior:**
- Cooldown period: Default 5 minutes between scaling actions
- Metrics evaluated every 1 minute
- Time window: Look back period for averaging (typically 5-10 minutes)
- Multiple rules evaluated independently

---

## PowerShell & CLI Commands

### VM Creation and Management

```powershell
# Get available VM sizes in region
Get-AzVMSize -Location "EastUS"

# Create VM with basic settings
New-AzVM `
    -ResourceGroupName "myResourceGroup" `
    -Name "myVM" `
    -Location "EastUS" `
    -ImageName "UbuntuLTS" `
    -Size "Standard_B2s" `
    -PublicIpAddressName "myPublicIP" `
    -OpenPorts 22, 80, 443

# Get VM details
Get-AzVM -ResourceGroupName "myResourceGroup" -Name "myVM" | Format-Table

# Start/Stop/Deallocate VM
Start-AzVM -ResourceGroupName "myResourceGroup" -Name "myVM"
Stop-AzVM -ResourceGroupName "myResourceGroup" -Name "myVM"
Disconnect-AzVM -ResourceGroupName "myResourceGroup" -Name "myVM" # Deallocate

# Resize VM
Stop-AzVM -ResourceGroupName "myResourceGroup" -Name "myVM"
$vm = Get-AzVM -ResourceGroupName "myResourceGroup" -Name "myVM"
$vm.HardwareProfile.VmSize = "Standard_D2s_v3"
Update-AzVM -ResourceGroupName "myResourceGroup" -VM $vm
Start-AzVM -ResourceGroupName "myResourceGroup" -Name "myVM"

# Remove VM
Remove-AzVM -ResourceGroupName "myResourceGroup" -Name "myVM"
```

```bash
# Get VM sizes
az vm list-sizes --location eastus

# Create VM
az vm create \
    --resource-group myResourceGroup \
    --name myVM \
    --image UbuntuLTS \
    --size Standard_B2s \
    --admin-username azureuser \
    --admin-password MyPassword@123

# Get VM info
az vm show --resource-group myResourceGroup --name myVM

# Start/Stop/Deallocate
az vm start --resource-group myResourceGroup --name myVM
az vm stop --resource-group myResourceGroup --name myVM
az vm deallocate --resource-group myResourceGroup --name myVM

# Resize VM
az vm resize --resource-group myResourceGroup --name myVM --size Standard_D2s_v3

# Delete VM
az vm delete --resource-group myResourceGroup --name myVM
```

### Disk Operations

```powershell
# Create managed disk
$diskConfig = New-AzDiskConfig `
    -Location "EastUS" `
    -DiskSizeGB 256 `
    -StorageAccountType "Premium_LRS"

New-AzDisk `
    -ResourceGroupName "myResourceGroup" `
    -DiskName "myDataDisk" `
    -Disk $diskConfig

# List disks
Get-AzDisk -ResourceGroupName "myResourceGroup"

# Attach disk to VM
$vm = Get-AzVM -ResourceGroupName "myResourceGroup" -Name "myVM"
$disk = Get-AzDisk -ResourceGroupName "myResourceGroup" -Name "myDataDisk"
Add-AzVMDataDisk -VM $vm -Name "myDataDisk" -ManagedDiskId $disk.Id -Lun 0
Update-AzVM -ResourceGroupName "myResourceGroup" -VM $vm

# Detach disk
Remove-AzVMDataDisk -VM $vm -DataDiskNames "myDataDisk"
Update-AzVM -ResourceGroupName "myResourceGroup" -VM $vm

# Convert disk type (Premium to Standard)
$disk = Get-AzDisk -ResourceGroupName "myResourceGroup" -Name "myDataDisk"
$disk.Sku = [Microsoft.Azure.Management.Compute.Models.DiskSku]::new("StandardSSD_LRS")
Update-AzDisk -ResourceGroupName "myResourceGroup" -Disk $disk -DiskName $disk.Name
```

```bash
# Create managed disk
az disk create \
    --resource-group myResourceGroup \
    --name myDataDisk \
    --size-gb 256 \
    --sku Premium_LRS

# List disks
az disk list --resource-group myResourceGroup

# Attach disk
az vm disk attach \
    --resource-group myResourceGroup \
    --vm-name myVM \
    --name myDataDisk

# Detach disk
az vm disk detach \
    --resource-group myResourceGroup \
    --vm-name myVM \
    --name myDataDisk

# Convert disk type
az disk update \
    --resource-group myResourceGroup \
    --name myDataDisk \
    --sku StandardSSD_LRS
```

### Disk Encryption

```powershell
# Enable encryption at host (new VM)
New-AzVMConfig -VMName "myVM" -VMSize "Standard_D2s_v2" -EncryptionAtHost

# Enable encryption at host (existing VM)
Update-AzVM `
    -ResourceGroupName "myResourceGroup" `
    -VM (Get-AzVM -ResourceGroupName "myResourceGroup" -Name "myVM") `
    -EncryptionAtHost

# Check encryption status
Get-AzVM -ResourceGroupName "myResourceGroup" -Name "myVM" | Select-Object EncryptionAtHostEnabled

# Enable Azure Disk Encryption (before retirement)
Enable-AzVMDiskEncryption `
    -ResourceGroupName "myResourceGroup" `
    -VMName "myVM" `
    -DiskEncryptionKeyVaultUrl "https://myKeyVault.vault.azure.net/" `
    -DiskEncryptionKeyVaultId (Get-AzKeyVault -VaultName "myKeyVault").ResourceId `
    -VolumeType "All"

# Check ADE status
Get-AzVMDiskEncryptionStatus `
    -ResourceGroupName "myResourceGroup" `
    -VMName "myVM"

# Disable encryption
Disable-AzVMDiskEncryption `
    -ResourceGroupName "myResourceGroup" `
    -VMName "myVM" `
    -VolumeType "All"
```

```bash
# Enable encryption at host
az vm update \
    --resource-group myResourceGroup \
    --name myVM \
    --encryption-at-host true

# Check encryption status
az vm show \
    --resource-group myResourceGroup \
    --name myVM \
    --query "securityProfile.encryptionAtHost"

# Create disk encryption set
az disk-encryption-set create \
    --resource-group myResourceGroup \
    --name myDES \
    --key-url "https://myKeyVault.vault.azure.net/keys/myKey/xyz123"
```

### Availability and Scaling

```powershell
# Create availability set
New-AzAvailabilitySet `
    -ResourceGroupName "myResourceGroup" `
    -Name "myAvailabilitySet" `
    -Location "EastUS"

# Create VMSS
New-AzVmss `
    -ResourceGroupName "myResourceGroup" `
    -VMScaleSetName "myVMSS" `
    -VMSize "Standard_D2s_v3" `
    -InstanceCount 3

# Get VMSS
Get-AzVmss -ResourceGroupName "myResourceGroup"

# Scale VMSS
Update-AzVmss `
    -ResourceGroupName "myResourceGroup" `
    -VMScaleSetName "myVMSS" `
    -SkipVMProfile `
    -Capacity 5

# Delete VMSS
Remove-AzVmss `
    -ResourceGroupName "myResourceGroup" `
    -VMScaleSetName "myVMSS"
```

```bash
# Create availability set
az vm availability-set create \
    --resource-group myResourceGroup \
    --name myAvailabilitySet

# Create VMSS
az vmss create \
    --resource-group myResourceGroup \
    --name myVMSS \
    --image UbuntuLTS \
    --instance-count 3 \
    --vm-sku Standard_D2s_v3

# List VMSS
az vmss list --resource-group myResourceGroup

# Scale VMSS
az vmss scale \
    --resource-group myResourceGroup \
    --name myVMSS \
    --new-capacity 5

# Delete VMSS
az vmss delete \
    --resource-group myResourceGroup \
    --name myVMSS
```

---

## Exam Tips & Key Concepts

### Critical Concepts for AZ-104

1. **VM Sizing Selection**
   - Match workload characteristics to family (Compute/Memory/Storage optimized)
   - Premium Storage capable = 's' in size name
   - Use SKU selector tool for complex requirements
   - Can only resize within same family in most cases

2. **Managed Disks**
   - Always use managed disks (no unmanaged disks for new VMs)
   - Size determines performance for Premium/Standard SSDs
   - Can change disk type (Premium ↔ Standard SSD) but requires downtime
   - Ultra Disks and Premium SSD v2 cannot be OS disks
   - Standard HDD retiring Sept 8, 2028 as OS disk

3. **Disk Encryption**
   - Encryption at Host is the future (Azure Disk Encryption retiring Sept 15, 2028)
   - Server-side encryption always enabled automatically
   - Temp disks (D:) only encrypted with Encryption at Host
   - Customer-managed keys require Azure Key Vault setup
   - No performance impact with encryption

4. **Availability for High Availability**
   - Availability Set: 99.95% SLA (single region protection)
   - Availability Zones: 99.99% SLA (datacenter-level protection)
   - Use zones over sets when available (better resilience)
   - Cannot mix zones and sets on same VM
   - Availability set limits: 3 FDs, 20 UDs per region

5. **Virtual Machine Scale Sets**
   - Flexible orchestration supports up to 1,000 VMs (vs 600 for Uniform)
   - Flexible allows mixed sizes/images and Spot + On-Demand
   - Always use load balancer for traffic distribution
   - Autoscale rules evaluated every 1 minute
   - Cool-down period prevents rapid scaling up/down

6. **Vertical vs. Horizontal Scaling**
   - Vertical (resize): requires downtime, limited by size availability
   - Horizontal (VMSS): no downtime, provides redundancy, recommended
   - Azure supports both but horizontal is enterprise best practice

### Exam Question Patterns

- **"Choose the most cost-effective..."** → Check disk size, Standard SSD vs Premium, correct family
- **"Ensure 99.99% availability..."** → Availability Zones (never just Availability Set)
- **"Minimize downtime for upgrade..."** → Horizontal scaling with VMSS, rolling updates
- **"Comply with encryption requirements..."** → Encryption at Host for new VMs, ADE migration path
- **"High-performance database..."** → Memory-optimized with Premium/Ultra disk, zone redundancy
- **"IoT sensors ingesting data..."** → Storage optimized with high throughput, Standard SSD/HDD

---

## Exam Scenarios

### Scenario 1: Design High-Performance Database Infrastructure

**Situation:**
You need to host a SQL Server database handling 10,000 transactions per second. The database is 2 TB and must have 99.99% availability with sub-10ms latency.

**Requirements Analysis:**
- High IOPS and throughput → Premium SSD v2 or Ultra Disk
- Large dataset (2 TB) → Premium SSD v2 or larger Premium SSD
- 99.99% SLA → Availability Zones (minimum 2 zones)
- Memory-intensive → E-series or M-series VMs
- Sub-10ms latency → Zones within same region (natural latency <2ms)

**Solution:**
1. Deploy 2 VMs in different availability zones (Zone 1, Zone 2)
2. VM size: Standard_E8s_v5 (8 vCPU, 64 GB RAM) - memory optimized
3. OS disk: Premium SSD P20 (500 GiB, 2,300 IOPS)
4. Data disk: Premium SSD v2 (2 TiB) with 80,000 IOPS capacity
5. Use SQL Server Always-On Availability Group across zones
6. Azure Load Balancer Standard SKU for health probes
7. Enable Encryption at Host for compliance
8. Implement zone-redundant storage (ZRS) for log backups

**Why this works:**
- Premium SSD v2 provides 80,000 IOPS for transaction throughput
- E-series with 64 GB provides memory for working set
- Zones provide 99.99% SLA and datacenter failure protection
- Always-On across zones ensures failover capability
- Sub-10ms latency maintained within region

---

### Scenario 2: Cost-Optimize Development/Test Environment

**Situation:**
Development team needs 20 VMs for testing web applications. Current monthly cost is $8,000. Need to reduce by 50% without affecting performance for 8-hour workday testing.

**Current Setup:**
- 20 x Standard_D2s_v3 (2 vCPU, 8 GB RAM)
- Premium SSD disks for OS
- Always running 24/7
- No autoscaling

**Solution:**
1. Migrate to Standard_B2s (burstable, lower base cost)
2. Switch OS disks to Standard SSD (cheaper than Premium)
3. Implement VM shutdown schedule (stop VMs outside 8-hour window)
4. Use Reserved Instances for 20 VMs (30-40% savings)
5. Deallocate VMs when not in use (no compute charges)
6. Use single availability set (not zones, as cost is factor)

**Cost Calculation:**
- Original: 20 × Standard_D2s_v3 × 730 hours = ~$5,840
- Plus Premium SSD: ~$2,160
- **Total: ~$8,000/month**

- Optimized: 20 × Standard_B2s × 240 hours (8hr/day) = ~$576
- Plus Standard SSD: ~$400
- Reserved instance (1-year): ~$3,200 + Runtime use
- **Total: ~$4,000-4,200/month (52% reduction)**

---

### Scenario 3: Deploy Application with Automatic Load Balancing

**Situation:**
E-commerce website traffic varies from 100 to 5,000 requests per second throughout the day and week. Currently using fixed infrastructure, oversized to handle peak load.

**Requirements:**
- Auto-scale from 3 to 30 instances based on load
- Average CPU < 50% during normal operation
- Scale-out when CPU > 70% for 5 minutes
- Scale-in when CPU < 30% for 5 minutes
- High availability across fault domains
- 5-minute cooldown between scaling actions

**Solution:**
1. Create Virtual Machine Scale Set with Flexible orchestration
2. VM size: Standard_B2s (burstable, cost-effective)
3. Instance count: 3 minimum, 30 maximum
4. Load balancer: Azure Load Balancer Standard
5. Configure autoscale rules:
   - Scale-out: CPU > 70% over 5-minute average, add 2 instances
   - Scale-in: CPU < 30% over 5-minute average, remove 1 instance
   - Cooldown: 5 minutes between actions
6. OS disk: Standard SSD (sufficient for web tier)
7. Data disk: Not needed (stateless application)
8. Deploy across availability zones (2 or 3) for resilience

**PowerShell for autoscale:**
```powershell
# Create scale-out rule for CPU > 70%
$scaleOut = New-AzAutoscaleRule `
    -MetricName "Percentage CPU" `
    -MetricResourceId $vmss.Id `
    -Operator GreaterThan `
    -Threshold 70 `
    -TimeWindow ([System.TimeSpan]::FromMinutes(5)) `
    -TimeGrain ([System.TimeSpan]::FromMinutes(1)) `
    -ScaleActionDirection Increase `
    -ScaleActionValue 2

# Create scale-in rule for CPU < 30%
$scaleIn = New-AzAutoscaleRule `
    -MetricName "Percentage CPU" `
    -MetricResourceId $vmss.Id `
    -Operator LessThan `
    -Threshold 30 `
    -TimeWindow ([System.TimeSpan]::FromMinutes(5)) `
    -TimeGrain ([System.TimeSpan]::FromMinutes(1)) `
    -ScaleActionDirection Decrease `
    -ScaleActionValue 1 `
    -ScaleActionCooldown ([System.TimeSpan]::FromMinutes(5))

# Apply rules
$profile = New-AzAutoscaleProfile `
    -Name "LoadBalancedProfile" `
    -DefaultCapacity 3 `
    -MinimumCapacity 3 `
    -MaximumCapacity 30 `
    -Rule $scaleOut, $scaleIn

New-AzAutoscaleSetting `
    -Name "WebAppAutoscale" `
    -ResourceGroupName "myResourceGroup" `
    -TargetResourceId $vmss.Id `
    -AutoscaleProfile $profile `
    -Location "EastUS"
```

---

### Scenario 4: Encrypt Existing Production VMs

**Situation:**
Your organization has compliance requirement to encrypt all production VMs by end of quarter. You have 50 legacy VMs using platform-managed encryption. Management wants minimal downtime.

**Situation Details:**
- Current: Platform-managed keys (default SSE)
- Requirement: Customer-managed keys for audit trail
- Target: Encryption at Host (future-proof against ADE retirement)
- Constraint: Minimal downtime, staggered deployment

**Migration Path:**
1. Phase 1 (Week 1-2): Create Key Vault and Disk Encryption Set
   - Single Key Vault per resource group (can be shared)
   - Create rotation policy for annual key rotation
   
2. Phase 2 (Week 3-6): Migrate VMs in waves (10 VMs per week)
   - Stop VM (deallocate to save money while stopped)
   - Detach OS and data disks
   - Create new disks from snapshots with encryption set
   - Attach new disks to VM
   - Start VM
   - Downtime: ~10-15 minutes per VM
   
3. Phase 3 (Week 7-8): Enable Encryption at Host on new VMs
   - New VMs use Encryption at Host by default
   - Update VM images to include latest OS patches

**Implementation (First VM):**
```powershell
# 1. Create Key Vault and Encryption Set
$rgName = "myResourceGroup"
$keyVaultName = "myKeyVault"
$desName = "myDES"

# Create Key Vault
$keyVault = New-AzKeyVault `
    -VaultName $keyVaultName `
    -ResourceGroupName $rgName `
    -Location "EastUS" `
    -EnabledForDiskEncryption

# Create key
$key = Add-AzKeyVaultKey `
    -VaultName $keyVaultName `
    -Name "diskEncryptionKey" `
    -Destination Software

# Create Disk Encryption Set
$des = New-AzDiskEncryptionSet `
    -ResourceGroupName $rgName `
    -Name $desName `
    -KeyUrl $key.Key.Kid `
    -SourceVaultId $keyVault.ResourceId

# 2. Migrate existing VM
$vmName = "productionVM1"
$vm = Get-AzVM -ResourceGroupName $rgName -Name $vmName

# Stop VM
Stop-AzVM -ResourceGroupName $rgName -Name $vmName

# Get current disk IDs
$osDiskId = $vm.StorageProfile.OsDisk.ManagedDisk.Id
$dataDisks = $vm.StorageProfile.DataDisks

# Create snapshot of OS disk
$snapshotConfig = New-AzSnapshotConfig `
    -SourceUri $osDiskId `
    -CreateOption Copy `
    -Location "EastUS"
$snapshot = New-AzSnapshot `
    -ResourceGroupName $rgName `
    -SnapshotName "snapshot-$vmName-os" `
    -Snapshot $snapshotConfig

# Create encrypted disk from snapshot
$newDiskConfig = New-AzDiskConfig `
    -Location "EastUS" `
    -CreateOption Copy `
    -SourceResourceId $snapshot.Id `
    -EncryptionSettingsCollection New-AzDiskEncryptionSetCollection -EncryptionSettingsElementId $des.Id

$newOSDisk = New-AzDisk `
    -ResourceGroupName $rgName `
    -DiskName "osdisk-$vmName-encrypted" `
    -Disk $newDiskConfig

# Detach old disk
Remove-AzVMOSDisk -VM $vm
Update-AzVM -ResourceGroupName $rgName -VM $vm

# Attach encrypted disk
Set-AzVMOSDisk `
    -VM $vm `
    -ManagedDiskId $newOSDisk.Id `
    -CreateOption Attach `
    -Windows

Update-AzVM -ResourceGroupName $rgName -VM $vm

# Start VM
Start-AzVM -ResourceGroupName $rgName -Name $vmName
```

---

### Scenario 5: Data Disk Performance Bottleneck

**Situation:**
Your SQL Server VM is experiencing slow query performance. Monitoring shows disk wait times > 100ms and throughput at 500 MB/s (limit). Database is 800 GB, growing 20 GB/month. Current setup uses Premium SSD P30 (1 TiB).

**Diagnosis:**
- Current disk: Premium SSD P30 = 5,000 IOPS, 200 MB/s throughput
- Query workload: 50 concurrent connections
- Wait stats: 80% disk I/O wait
- Growth projection: Will reach capacity in ~11 months

**Solution Options:**

**Option A: Upgrade to Premium SSD v2** (Recommended)
- Premium SSD v2 with 2 TiB = 80,000 IOPS, 1,200 MB/s
- Cost: ~$180/month (vs $130 for P30)
- Benefit: 16x IOPS, 6x throughput, future growth covered

**Option B: Multiple disks with striping**
- Add 3 x Premium SSD P30 disks in RAID-0
- Total: 15,000 IOPS, 600 MB/s (insufficient for growth)
- Cost: ~$130 × 3 = $390/month (exceeds Option A)
- Complexity: Requires RAID software configuration

**Option C: Ultra Disk**
- Single Ultra Disk configured for 80,000 IOPS
- Cost: ~$200/month + per-IOPS charges
- Benefit: Maximum flexibility, highest performance
- Drawback: Only supported on certain VM sizes, no replication

**Recommended: Option A**
1. Stop VM
2. Create snapshot of current P30 disk
3. Create Premium SSD v2 disk (2 TiB) from snapshot
4. Detach P30, attach v2
5. Start VM (full retest)
6. Verify performance improvement
7. Downtime: 10-15 minutes

**Capacity Planning:**
- Current: 800 GB
- Growth: 20 GB/month
- Premium SSD v2 2 TiB: Safe for ~60 months (5 years)
- Review annually or if growth rate increases

---

## Troubleshooting Guide

### Issue 1: Cannot Resize VM

**Symptoms:**
- Error: "The requested size is not available in the current region"
- Or: "VM has constraints that prevent resizing"

**Causes:**
- Size not available in target region
- Trying to resize to incompatible family
- VM in scale set (different resize process)
- Old VM size not supported for new target size

**Solutions:**

1. **Check available sizes in region:**
```powershell
Get-AzVMSize -Location "EastUS" | Where-Object {$_.Name -like "Standard_D*"}
```

2. **Deallocate VM first (mandatory for some resizes):**
```powershell
Stop-AzVM -ResourceGroupName "myResourceGroup" -Name "myVM"
Update-AzVM -ResourceGroupName "myResourceGroup" -VM $vm
```

3. **Check size compatibility:**
```powershell
# Resizing within same family generally works
# Standard_D2s_v3 → Standard_D4s_v3 (same family) = OK
# Standard_D2s_v3 → Standard_E2s_v3 (different family) = May fail
```

4. **If in scale set, update scale set instead:**
```powershell
Update-AzVmss `
    -ResourceGroupName "myResourceGroup" `
    -VMScaleSetName "myVMSS" `
    -SkipVMProfile `
    -VirtualMachineProfile $vmss.VirtualMachineProfile
```

---

### Issue 2: Disk Encryption Fails

**Symptoms:**
- Enable-AzVMDiskEncryption returns error
- Encryption operation hangs

**Common Causes:**
- Key Vault not enabled for disk encryption
- VM in unsupported configuration (L-series, Write Accelerator)
- Insufficient Key Vault permissions
- Network connectivity to Key Vault blocked

**Solutions:**

1. **Verify Key Vault configuration:**
```powershell
# Check if enabled for disk encryption
$kv = Get-AzKeyVault -VaultName "myKeyVault"
$kv.EnabledForDiskEncryption # Should be True

# If False, enable it
Set-AzKeyVaultAccessPolicy -VaultName "myKeyVault" -EnabledForDiskEncryption
```

2. **Check VM compatibility:**
```powershell
# L-series, M-series with Write Accelerator, and others NOT supported
$vm = Get-AzVM -ResourceGroupName "myResourceGroup" -Name "myVM"
$vm.HardwareProfile.VmSize # Check against restrictions list
```

3. **Enable encryption with correct parameters:**
```powershell
# Ensure Key Vault is in same region
# For VolumeType: "OS", "Data", or "All"
# "All" requires separate key vault in some cases

Enable-AzVMDiskEncryption `
    -ResourceGroupName "myResourceGroup" `
    -VMName "myVM" `
    -DiskEncryptionKeyVaultUrl (Get-AzKeyVault -VaultName "myKeyVault").VaultUri `
    -DiskEncryptionKeyVaultId (Get-AzKeyVault -VaultName "myKeyVault").ResourceId `
    -VolumeType "OS" `
    -Force
```

4. **Use Encryption at Host instead (recommended for new VMs):**
```powershell
# Simpler, no Key Vault dependency, encrypts temp disks too
Update-AzVM -ResourceGroupName "myResourceGroup" -VM $vm -EncryptionAtHost
```

---

### Issue 3: Autoscale Not Triggering

**Symptoms:**
- VMSS not scaling despite high CPU
- Always at minimum capacity
- Autoscale rules not executing

**Common Causes:**
- Metric threshold never reached (workload too light)
- Cooldown period preventing scale actions
- Incorrect metric name or resource ID
- Insufficient minimum capacity set

**Solutions:**

1. **Verify autoscale configuration:**
```powershell
# Get autoscale settings
Get-AzAutoscaleSetting `
    -ResourceGroupName "myResourceGroup" `
    -Name "myAutoscaleSetting"

# Check profiles and rules
$setting = Get-AzAutoscaleSetting -ResourceGroupName "myResourceGroup" -Name "myAutoscaleSetting"
$setting.Profiles | ForEach-Object { $_.Rules }
```

2. **Test metric threshold:**
```bash
# SSH to VM and generate CPU load for 6+ minutes
# Watch: https://learn.microsoft.com/en-us/cli/azure/monitor/metrics
az monitor metrics list \
    --resource-group myResourceGroup \
    --resource myVMSS \
    --resource-type "Microsoft.Compute/virtualMachineScaleSets" \
    --metric "Percentage CPU" \
    --interval PT1M
```

3. **Adjust thresholds if needed:**
```powershell
# Lower threshold for easier scaling
$rule.MetricTrigger.Threshold = 60 # From 70%

# Extend time window to be more stable
$rule.MetricTrigger.TimeWindow = [System.TimeSpan]::FromMinutes(10)
```

4. **Check cooldown period:**
```powershell
# Default 5-minute cooldown prevents rapid scaling
# If scaling frequently, may be stuck in cooldown
$rule.ScaleActionCooldown # Check this value
```

5. **Ensure minimum capacity reasonable:**
```powershell
$profile.MinCapacity # Should be >= 1
# If set to 0, scale-in may remove all instances
```

---

### Issue 4: High Disk I/O Latency

**Symptoms:**
- Slow application performance
- Disk queue length high
- I/O wait times > 50ms

**Causes:**
- Undersized disk (not enough IOPS/throughput)
- Disk throttling due to VM IOPS limits
- Excessive random I/O not optimized for disk type
- Multiple VMs competing for single disk

**Solutions:**

1. **Check current disk performance:**
```powershell
$disk = Get-AzDisk -ResourceGroupName "myResourceGroup" -Name "myDataDisk"
$disk.Sku # Check current SKU (P10, P30, v2, etc.)

# Check IOPS and throughput limits
# Premium SSD P10 = 500 IOPS, 100 MB/s
# Premium SSD P80 = 20,000 IOPS, 900 MB/s
# Premium SSD v2 = up to 80,000 IOPS, 1,200 MB/s
```

2. **Upgrade disk type:**
```powershell
# Switch from P10 to P30
$disk.Sku = [Microsoft.Azure.Management.Compute.Models.DiskSku]::new("Premium_LRS")
# But resize GiB first since P30 = 1 TiB

# Better: Use Premium SSD v2 for best performance
$disk.Sku = [Microsoft.Azure.Management.Compute.Models.DiskSku]::new("PremiumV2_LRS")
Update-AzDisk -ResourceGroupName "myResourceGroup" -Disk $disk -DiskName $disk.Name
```

3. **Check VM IOPS limits:**
```powershell
# Each VM size has max IOPS limit
# Standard_D2s_v3: 4,800 IOPS max
# Standard_D4s_v3: 9,600 IOPS max
# Standard_E8s_v5: 12,800 IOPS max

# If disk IOPS exceeds VM limit, upgrade VM size
Get-AzVMSize -Location "EastUS" | Select-Object Name, MemoryInMB
```

4. **Add additional disks for striping (if appropriate):**
```powershell
# Create 4 x P30 disks in RAID-0
# Aggregate: 20,000 IOPS, 800 MB/s
# Configure in Windows via Disk Management or Linux via mdadm
```

---

### Issue 5: VM Won't Start After Disk Encryption

**Symptoms:**
- VM stuck in "starting" state
- Error about bitlocker or encryption
- Boot timeout

**Causes:**
- Encryption operation incomplete
- Key Vault permissions changed
- Encrypted disk corrupted
- BitLocker recovery key needed

**Solutions:**

1. **Check encryption status:**
```powershell
Get-AzVMDiskEncryptionStatus `
    -ResourceGroupName "myResourceGroup" `
    -VMName "myVM"

# Look for error messages in Status field
```

2. **Wait for encryption to complete:**
```powershell
# Encryption can take 30+ minutes for large disks
# Check progress every 5 minutes
# Do NOT restart during encryption

# Check VM extension logs
Get-AzVMExtension -ResourceGroupName "myResourceGroup" -VMName "myVM"
```

3. **If stuck, attempt recovery:**
```powershell
# Disable encryption gracefully
Disable-AzVMDiskEncryption `
    -ResourceGroupName "myResourceGroup" `
    -VMName "myVM" `
    -VolumeType "All" `
    -Force

# Wait for decryption to complete
# Then try encryption again if needed
```

4. **Last resort: Restore from snapshot:**
```powershell
# If VM cannot recover, restore from snapshot
$snapshot = Get-AzSnapshot -ResourceGroupName "myResourceGroup" -SnapshotName "pre-encryption-snapshot"
$diskConfig = New-AzDiskConfig -CreateOption Copy -SourceResourceId $snapshot.Id
$disk = New-AzDisk -Disk $diskConfig -DiskName "recovered-disk" -ResourceGroupName "myResourceGroup"

# Create new VM with recovered disk
# Or restore disk to existing VM after detaching corrupted disk
```

---

## Related Topics

- **Deploy VMs with ARM Templates**: Infrastructure as Code approach for complex deployments
- **Manage Azure Identities**: RBAC for VM management, managed identities for VM access
- **Virtual Networking**: Network interfaces, subnets, NSGs for VM connectivity
- **Azure Storage**: Storage accounts for VM data, managed disk accounts
- **Monitoring**: Azure Monitor for VM performance metrics and diagnostics

---

## Key References

- [Azure Virtual Machines Documentation](https://learn.microsoft.com/en-us/azure/virtual-machines/)
- [Azure VM Sizes](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/overview)
- [Azure Managed Disks](https://learn.microsoft.com/en-us/azure/virtual-machines/managed-disks-overview)
- [Disk Encryption Guide](https://learn.microsoft.com/en-us/azure/virtual-machines/disk-encryption)
- [Virtual Machine Scale Sets](https://learn.microsoft.com/en-us/azure/virtual-machine-scale-sets/overview)
- [Availability Sets and Zones](https://learn.microsoft.com/en-us/azure/virtual-machines/availability)
