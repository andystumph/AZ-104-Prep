# AZ-104 Quick Reference Guide

> **Fast reference for Azure Administrator exam - Commands, concepts, and key facts**

## Table of Contents
- [PowerShell Quick Reference](#powershell-quick-reference)
- [Azure CLI Quick Reference](#azure-cli-quick-reference)
- [Service Limits and Quotas](#service-limits-and-quotas)
- [Key Concepts Cheat Sheet](#key-concepts-cheat-sheet)
- [Port Numbers Reference](#port-numbers-reference)
- [SKU and Pricing Tiers](#sku-and-pricing-tiers)
- [Exam Day Checklist](#exam-day-checklist)

---

## PowerShell Quick Reference

### Connection and Context
```powershell
# Connect to Azure
Connect-AzAccount

# List subscriptions
Get-AzSubscription

# Set active subscription
Set-AzContext -SubscriptionId "subscription-id"
Set-AzContext -SubscriptionName "subscription-name"

# Get current context
Get-AzContext

# Save context for later use
Save-AzContext -Path "context.json"
Import-AzContext -Path "context.json"
```

### Resource Groups
```powershell
# Create resource group
New-AzResourceGroup -Name "MyRG" -Location "eastus"

# List resource groups
Get-AzResourceGroup

# Get specific resource group
Get-AzResourceGroup -Name "MyRG"

# Remove resource group
Remove-AzResourceGroup -Name "MyRG" -Force

# List resources in RG
Get-AzResource -ResourceGroupName "MyRG"

# Move resources
Move-AzResource -DestinationResourceGroupName "DestRG" -ResourceId "/subscriptions/.../resourceGroups/SourceRG/providers/..."
```

### Virtual Machines
```powershell
# Create VM (quick)
New-AzVM -ResourceGroupName "MyRG" -Name "MyVM" -Location "eastus" -Image "Win2022Datacenter"

# List VMs
Get-AzVM

# Get VM details
Get-AzVM -ResourceGroupName "MyRG" -Name "MyVM"

# Start/Stop/Restart VM
Start-AzVM -ResourceGroupName "MyRG" -Name "MyVM"
Stop-AzVM -ResourceGroupName "MyRG" -Name "MyVM" -Force
Restart-AzVM -ResourceGroupName "MyRG" -Name "MyVM"

# Get VM status
Get-AzVM -ResourceGroupName "MyRG" -Name "MyVM" -Status

# Resize VM
$vm = Get-AzVM -ResourceGroupName "MyRG" -Name "MyVM"
$vm.HardwareProfile.VmSize = "Standard_DS3_v2"
Update-AzVM -VM $vm -ResourceGroupName "MyRG"

# Remove VM
Remove-AzVM -ResourceGroupName "MyRG" -Name "MyVM" -Force
```

### Storage Accounts
```powershell
# Create storage account
New-AzStorageAccount -ResourceGroupName "MyRG" -Name "mystorageacct" -Location "eastus" -SkuName "Standard_LRS"

# Get storage account
Get-AzStorageAccount -ResourceGroupName "MyRG" -Name "mystorageacct"

# List storage accounts
Get-AzStorageAccount

# Get storage account keys
Get-AzStorageAccountKey -ResourceGroupName "MyRG" -Name "mystorageacct"

# Regenerate key
New-AzStorageAccountKey -ResourceGroupName "MyRG" -Name "mystorageacct" -KeyName "key1"

# Get context
$ctx = (Get-AzStorageAccount -ResourceGroupName "MyRG" -Name "mystorageacct").Context

# Create container
New-AzStorageContainer -Name "mycontainer" -Context $ctx -Permission Blob

# Upload blob
Set-AzStorageBlobContent -File "file.txt" -Container "mycontainer" -Blob "file.txt" -Context $ctx
```

### Virtual Networks
```powershell
# Create VNet
$subnet = New-AzVirtualNetworkSubnetConfig -Name "Subnet1" -AddressPrefix "10.0.1.0/24"
New-AzVirtualNetwork -Name "MyVNet" -ResourceGroupName "MyRG" -Location "eastus" -AddressPrefix "10.0.0.0/16" -Subnet $subnet

# Get VNet
Get-AzVirtualNetwork -ResourceGroupName "MyRG" -Name "MyVNet"

# Add subnet
$vnet = Get-AzVirtualNetwork -ResourceGroupName "MyRG" -Name "MyVNet"
Add-AzVirtualNetworkSubnetConfig -Name "Subnet2" -VirtualNetwork $vnet -AddressPrefix "10.0.2.0/24"
$vnet | Set-AzVirtualNetwork

# Create VNet peering
Add-AzVirtualNetworkPeering -Name "VNet1-to-VNet2" -VirtualNetwork $vnet1 -RemoteVirtualNetworkId $vnet2.Id

# Create NSG
New-AzNetworkSecurityGroup -Name "MyNSG" -ResourceGroupName "MyRG" -Location "eastus"

# Add NSG rule
$nsg = Get-AzNetworkSecurityGroup -ResourceGroupName "MyRG" -Name "MyNSG"
Add-AzNetworkSecurityRuleConfig -NetworkSecurityGroup $nsg -Name "Allow-HTTP" -Priority 100 -Access Allow -Protocol Tcp -Direction Inbound -SourceAddressPrefix * -SourcePortRange * -DestinationAddressPrefix * -DestinationPortRange 80
$nsg | Set-AzNetworkSecurityGroup
```

### RBAC
```powershell
# List role definitions
Get-AzRoleDefinition

# Get specific role
Get-AzRoleDefinition "Contributor"

# List role assignments
Get-AzRoleAssignment
Get-AzRoleAssignment -SignInName "user@contoso.com"
Get-AzRoleAssignment -ResourceGroupName "MyRG"

# Assign role
New-AzRoleAssignment -SignInName "user@contoso.com" -RoleDefinitionName "Contributor" -ResourceGroupName "MyRG"

# Remove role assignment
Remove-AzRoleAssignment -SignInName "user@contoso.com" -RoleDefinitionName "Contributor" -ResourceGroupName "MyRG"

# Create custom role
New-AzRoleDefinition -InputFile "role.json"
```

### Azure Policy
```powershell
# Get policy definitions
Get-AzPolicyDefinition

# Get policy assignment
Get-AzPolicyAssignment

# Assign policy
New-AzPolicyAssignment -Name "Require-Tag" -PolicyDefinition $policyDef -Scope "/subscriptions/$subscriptionId"

# Remove policy assignment
Remove-AzPolicyAssignment -Name "Require-Tag" -Scope "/subscriptions/$subscriptionId"
```

---

## Azure CLI Quick Reference

### Connection and Context
```bash
# Login
az login

# List subscriptions
az account list --output table

# Set subscription
az account set --subscription "subscription-id"

# Show current subscription
az account show
```

### Resource Groups
```bash
# Create resource group
az group create --name MyRG --location eastus

# List resource groups
az group list --output table

# Show resource group
az group show --name MyRG

# Delete resource group
az group delete --name MyRG --yes --no-wait

# List resources in RG
az resource list --resource-group MyRG --output table
```

### Virtual Machines
```bash
# Create VM
az vm create --resource-group MyRG --name MyVM --image Win2022Datacenter --admin-username azureuser --admin-password 'P@ssw0rd1234!'

# List VMs
az vm list --output table

# Show VM
az vm show --resource-group MyRG --name MyVM

# Start/Stop/Restart VM
az vm start --resource-group MyRG --name MyVM
az vm stop --resource-group MyRG --name MyVM
az vm restart --resource-group MyRG --name MyVM

# Get VM status
az vm get-instance-view --resource-group MyRG --name MyVM

# Resize VM
az vm resize --resource-group MyRG --name MyVM --size Standard_DS3_v2

# Delete VM
az vm delete --resource-group MyRG --name MyVM --yes
```

### Storage Accounts
```bash
# Create storage account
az storage account create --name mystorageacct --resource-group MyRG --location eastus --sku Standard_LRS

# Show storage account
az storage account show --name mystorageacct --resource-group MyRG

# List storage accounts
az storage account list --output table

# Get connection string
az storage account show-connection-string --name mystorageacct --resource-group MyRG

# Get keys
az storage account keys list --resource-group MyRG --account-name mystorageacct

# Regenerate key
az storage account keys renew --resource-group MyRG --account-name mystorageacct --key primary

# Create container
az storage container create --name mycontainer --account-name mystorageacct

# Upload blob
az storage blob upload --account-name mystorageacct --container-name mycontainer --name file.txt --file file.txt
```

### Virtual Networks
```bash
# Create VNet
az network vnet create --resource-group MyRG --name MyVNet --address-prefix 10.0.0.0/16 --subnet-name Subnet1 --subnet-prefix 10.0.1.0/24

# List VNets
az network vnet list --output table

# Show VNet
az network vnet show --resource-group MyRG --name MyVNet

# Create subnet
az network vnet subnet create --resource-group MyRG --vnet-name MyVNet --name Subnet2 --address-prefix 10.0.2.0/24

# Create VNet peering
az network vnet peering create --name VNet1-to-VNet2 --resource-group MyRG --vnet-name VNet1 --remote-vnet VNet2 --allow-vnet-access

# Create NSG
az network nsg create --resource-group MyRG --name MyNSG

# Add NSG rule
az network nsg rule create --resource-group MyRG --nsg-name MyNSG --name Allow-HTTP --priority 100 --source-address-prefixes '*' --source-port-ranges '*' --destination-address-prefixes '*' --destination-port-ranges 80 --access Allow --protocol Tcp --direction Inbound
```

### RBAC
```bash
# List role definitions
az role definition list --output table

# Show role definition
az role definition list --name "Contributor"

# List role assignments
az role assignment list --output table
az role assignment list --assignee "user@contoso.com"

# Assign role
az role assignment create --assignee "user@contoso.com" --role "Contributor" --resource-group MyRG

# Remove role assignment
az role assignment delete --assignee "user@contoso.com" --role "Contributor" --resource-group MyRG

# Create custom role
az role definition create --role-definition role.json
```

---

## Service Limits and Quotas

### Subscription Limits
| Resource | Limit |
|----------|-------|
| Resource groups per subscription | 980 |
| Resources per resource group | 800 |
| Role assignments per subscription | 4,000 |
| Management groups per directory | 10,000 |
| Subscriptions per management group | Unlimited |
| Custom roles per directory | 5,000 |
| Tags per resource | 50 |
| Tag name length | 512 characters |
| Tag value length | 256 characters |

### Virtual Machine Limits
| Resource | Limit |
|----------|-------|
| VMs per availability set | 200 |
| Availability sets per subscription | 2,500 per region |
| VM scale set instances | 1,000 (default), up to 10,000 with orchestration mode |
| Data disks per VM | 64 (varies by size) |
| Max VM size (memory) | 12 TB (M series) |

### Storage Limits
| Resource | Limit |
|----------|-------|
| Storage accounts per subscription per region | 250 |
| Max capacity per storage account | 5 PiB |
| Blob containers per storage account | Unlimited |
| Blobs per container | Unlimited |
| Max blob size (block blob) | ~190.7 TiB |
| Max blob size (page blob) | 8 TiB |

### Network Limits
| Resource | Limit |
|----------|-------|
| Virtual networks per subscription | 1,000 |
| Subnets per VNet | 3,000 |
| VNet peerings per VNet | 500 |
| Public IP addresses per subscription | 1,000 |
| Private IP addresses per VNet | 65,536 |
| NSG rules per NSG | 1,000 |
| NSGs per subscription | 5,000 |

---

## Key Concepts Cheat Sheet

### Identity and Access

**Microsoft Entra ID vs Azure RBAC:**
- **Entra ID roles**: Manage Entra ID resources (users, groups, applications)
- **Azure RBAC roles**: Manage Azure resources (VMs, storage, networks)

**Common Roles:**
- **Owner**: Full access + can grant access
- **Contributor**: Full access but cannot grant access
- **Reader**: View only
- **User Access Administrator**: Manage user access only

### Storage

**Redundancy Options:**
- **LRS** (Locally Redundant): 3 copies in single datacenter
- **ZRS** (Zone Redundant): 3 copies across zones in region
- **GRS** (Geo Redundant): LRS + 3 copies in paired region
- **GZRS** (Geo-Zone Redundant): ZRS + 3 copies in paired region
- **RA-GRS/RA-GZRS**: Read access to secondary region

**Access Tiers:**
- **Hot**: Frequent access, higher storage cost, lower access cost
- **Cool**: Infrequent access (30+ days), lower storage cost, higher access cost
- **Cold**: Rare access (90+ days), lowest storage cost, highest access cost
- **Archive**: Long-term storage (180+ days), offline, rehydration required

**Blob Types:**
- **Block blobs**: Text and binary data (documents, images)
- **Append blobs**: Logging data (append only)
- **Page blobs**: VM disks, random access files

### Networking

**IP Address Types:**
- **Public**: Internet-accessible, can be static or dynamic
- **Private**: Internal VNet only, always static

**NSG vs Firewall:**
- **NSG**: Layer 4 (IP/Port), stateful, free
- **Azure Firewall**: Layer 3-7, threat intelligence, paid

**Service Endpoint vs Private Endpoint:**
- **Service Endpoint**: Routes traffic over Azure backbone, service is still public
- **Private Endpoint**: Injects private IP into your VNet, truly private

**VNet Peering:**
- Connects two VNets
- Non-transitive (must create peering between each pair)
- Can be in different regions (global peering)
- Traffic stays on Microsoft backbone

### Compute

**Availability Options:**
- **Availability Set**: Protects from hardware failures (99.95% SLA)
  - Update domains: Planned maintenance
  - Fault domains: Hardware failures
- **Availability Zone**: Protects from datacenter failures (99.99% SLA)
  - Separate physical locations in region
- **VM Scale Set**: Auto-scaling group of VMs

**Managed Disk Types:**
- **Ultra SSD**: Highest performance, sub-millisecond latency
- **Premium SSD**: Production workloads, consistent performance
- **Standard SSD**: Web servers, light workloads
- **Standard HDD**: Backup, non-critical workloads

### Monitoring

**Log Types:**
- **Activity Log**: Subscription-level events (who did what, when)
- **Resource Logs**: Resource-specific logs (sent to Log Analytics, Storage, Event Hub)
- **Metrics**: Numerical time-series data

**Alert Types:**
- **Metric Alerts**: Based on metrics (CPU > 80%)
- **Log Alerts**: Based on log queries
- **Activity Log Alerts**: Subscription events

---

## Port Numbers Reference

| Service | Port | Protocol |
|---------|------|----------|
| HTTP | 80 | TCP |
| HTTPS | 443 | TCP |
| SSH | 22 | TCP |
| RDP | 3389 | TCP |
| FTP | 21 | TCP |
| FTPS | 990 | TCP |
| SMTP | 25 | TCP |
| DNS | 53 | TCP/UDP |
| SQL Server | 1433 | TCP |
| MySQL | 3306 | TCP |
| PostgreSQL | 5432 | TCP |
| Redis | 6379 | TCP |
| MongoDB | 27017 | TCP |

---

## SKU and Pricing Tiers

### Storage Account
- **Standard (General Purpose v2)**: Most scenarios, blob/file/queue/table
- **Premium Block Blobs**: High transaction rates, low latency
- **Premium File Shares**: Enterprise file shares
- **Premium Page Blobs**: VM disks

### App Service
- **Free/Shared**: Dev/test, limited features
- **Basic**: Simple apps, manual scale
- **Standard**: Production apps, auto-scale, slots
- **Premium**: Enhanced performance, VNet integration
- **Isolated**: Dedicated environment, App Service Environment

### Load Balancer
- **Basic**: Free, limited features
- **Standard**: Paid, availability zones, more backends, diagnostics

### Public IP
- **Basic**: Dynamic allocation, no zone redundancy
- **Standard**: Static allocation, zone redundant

---

## Exam Day Checklist

### Before the Exam
- [ ] Valid ID ready (government-issued)
- [ ] Check exam appointment time and time zone
- [ ] Review [exam sandbox](https://aka.ms/examdemo)
- [ ] Quiet testing environment prepared (if online)
- [ ] Webcam and microphone tested (if online)
- [ ] Read [exam policies](https://learn.microsoft.com/en-us/credentials/certifications/certification-exam-policies)

### During the Exam
- [ ] Read each question completely
- [ ] Note time limit (120 minutes typical)
- [ ] Mark uncertain questions for review
- [ ] Look for keywords: "least", "except", "most", "first"
- [ ] Eliminate wrong answers first
- [ ] Don't overthink - go with your knowledge

### Question Types
1. **Multiple Choice**: Single or multiple correct answers
2. **Drag and Drop**: Arrange items in order or match
3. **Hot Area**: Click on part of an image
4. **Case Study**: Multiple questions about a scenario
5. **Build List**: Create a sequence
6. **Active Screen**: Perform task in simulated environment

### Common Traps
- Questions with "EXCEPT" or "NOT" in them
- Multiple valid answers but one is "BEST"
- Case studies with irrelevant information
- Technical terms used incorrectly
- Specific command syntax questions

### Time Management
- **15 min**: Review instructions and begin
- **90 min**: Complete all questions first pass
- **20 min**: Review marked questions
- **10 min**: Final review and ensure all answered
- **5 min**: Double-check case study answers

---

## Must-Know PowerShell Cmdlets

```powershell
# Connection
Connect-AzAccount
Set-AzContext

# Resources
New-AzResourceGroup
Get-AzResource

# VMs
New-AzVM
Get-AzVM
Start-AzVM / Stop-AzVM

# Storage
New-AzStorageAccount
Get-AzStorageAccountKey
New-AzStorageContainer

# Networking
New-AzVirtualNetwork
New-AzNetworkSecurityGroup
Add-AzVirtualNetworkPeering

# RBAC
Get-AzRoleDefinition
New-AzRoleAssignment
Get-AzRoleAssignment

# Policy
New-AzPolicyDefinition
New-AzPolicyAssignment

# Monitoring
New-AzMetricAlertRuleV2
New-AzActivityLogAlert
```

## Must-Know Azure CLI Commands

```bash
# Connection
az login
az account set

# Resources
az group create
az resource list

# VMs
az vm create
az vm list
az vm start / az vm stop

# Storage
az storage account create
az storage account keys list
az storage container create

# Networking
az network vnet create
az network nsg create
az network vnet peering create

# RBAC
az role definition list
az role assignment create
az role assignment list

# Policy
az policy definition create
az policy assignment create

# Monitoring
az monitor metrics alert create
az monitor activity-log alert create
```

---

**Remember:**
- Hands-on practice is crucial
- Understand concepts, not just commands
- Know when to use each service
- Practice in Azure Portal, PowerShell, and CLI
- Review weak areas multiple times

**Good luck on your AZ-104 exam!** 🎯
