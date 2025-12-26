# Secure Access to Virtual Networks

## Overview

Securing access to virtual networks is fundamental to protecting Azure resources. This guide covers Network Security Groups (NSGs), Application Security Groups (ASGs), Azure Bastion, and private connectivity options for safeguarding your virtual network infrastructure.

---

## Network Security Groups (NSGs)

### NSG Fundamentals

A Network Security Group (NSG) is a network-level firewall that controls traffic flow to and from resources in a virtual network.

**Key Characteristics:**
- Contains a list of security rules (Access Control List)
- Stateful packet filtering (remembers established connections)
- Evaluated based on rule priority (100-4096)
- Can be applied to subnets or network interfaces
- Default rules cannot be deleted but can be overridden
- Applied in direction order: Inbound rules first, then outbound

**Default Security Rules:**

| Name | Priority | Direction | Action | Source | Destination |
|------|----------|-----------|--------|--------|-------------|
| AllowVNetInBound | 65000 | Inbound | Allow | VirtualNetwork | VirtualNetwork |
| AllowAzureLoadBalancerInBound | 65001 | Inbound | Allow | AzureLoadBalancer | Any |
| DenyAllInBound | 65500 | Inbound | Deny | Any | Any |
| AllowVNetOutBound | 65000 | Outbound | Allow | VirtualNetwork | VirtualNetwork |
| AllowInternetOutBound | 65001 | Outbound | Allow | Any | Internet |
| DenyAllOutBound | 65500 | Outbound | Deny | Any | Any |

**Default Behavior:**
- All inbound traffic denied (except from VNet and Azure Load Balancer)
- All outbound traffic allowed (except explicit deny rules)
- VNet-to-VNet communication allowed by default

### Creating NSGs

**Azure Portal Method:**

1. Navigate to **Network security groups** > **+ Create**
2. Enter basic info (name, resource group, region)
3. Review default rules
4. Create
5. Add custom inbound/outbound rules

**PowerShell Method:**

```powershell
# Create NSG rule
$nsgRule = New-AzNetworkSecurityRuleConfig `
  -Name "AllowHTTP" `
  -Protocol Tcp `
  -Direction Inbound `
  -Priority 100 `
  -SourceAddressPrefix Internet `
  -SourcePortRange * `
  -DestinationAddressPrefix * `
  -DestinationPortRange 80 `
  -Access Allow

# Create NSG with rule
$nsg = New-AzNetworkSecurityGroup `
  -ResourceGroupName "myRG" `
  -Location "eastus" `
  -Name "myNSG" `
  -SecurityRules $nsgRule

# Associate NSG to subnet
$vnet = Get-AzVirtualNetwork -ResourceGroupName "myRG" -Name "myVNet"
$subnet = Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name "mySubnet"
$subnet.NetworkSecurityGroup = $nsg
Set-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -NetworkSecurityGroup $nsg | Set-AzVirtualNetwork
```

**Azure CLI Method:**

```bash
# Create NSG
az network nsg create \
  --resource-group myRG \
  --name myNSG

# Add rule
az network nsg rule create \
  --resource-group myRG \
  --nsg-name myNSG \
  --name AllowHTTP \
  --priority 100 \
  --source-address-prefixes Internet \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges 80 \
  --access Allow \
  --protocol Tcp \
  --direction Inbound

# Associate to subnet
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVNet \
  --name mySubnet \
  --network-security-group myNSG
```

### Security Rule Components

Each security rule consists of:

| Property | Values | Purpose |
|----------|--------|---------|
| **Name** | String (1-80 chars) | Unique rule identifier |
| **Priority** | 100-4096 | Evaluation order (lower = higher priority) |
| **Direction** | Inbound, Outbound | Traffic direction |
| **Access** | Allow, Deny | Permit or block traffic |
| **Protocol** | Tcp, Udp, Icmp, Esp, Any | Network protocol |
| **Source Address** | IP/CIDR, Service Tag, ASG, Any | Traffic source |
| **Source Port** | Port/Range/Any | Source port(s) |
| **Destination Address** | IP/CIDR, Service Tag, ASG, Any | Traffic destination |
| **Destination Port** | Port/Range/Any | Destination port(s) |

**Service Tags:**
Represent groups of IP addresses for Azure services; automatically updated.

Examples:
- `VirtualNetwork` - All VNet address space
- `Internet` - Public internet
- `AzureLoadBalancer` - Azure Load Balancer
- `Storage` - All storage accounts
- `Sql` - All SQL databases
- `AppService` - All App Service instances

**Source/Destination Options:**

| Type | Description | Example |
|------|-------------|---------|
| **IP Address** | Single IPv4/IPv6 | 203.0.113.0 |
| **CIDR Range** | Classless notation | 10.0.0.0/24 |
| **Any** | All addresses | * |
| **Service Tag** | Azure service group | Storage, Sql |
| **Application Security Group** | Logical VM grouping | asg-web |

### NSG Association Levels

**Subnet-Level:**
- Applied to all resources in subnet
- Simpler management
- Rules affect all VMs in subnet equally

**Network Interface Level:**
- Applied to specific VM NIC
- Granular control per VM
- Overrides subnet-level rules for that NIC

**Both Levels (Combined):**
- Effective rules = Subnet NSG rules + NIC NSG rules
- Both must allow traffic for successful communication

### NSG Best Practices

1. **Deny by Default, Allow by Exception**
   - Create explicit allow rules for required traffic
   - Minimize attack surface

2. **Use Service Tags**
   - Simplifies rule management
   - Automatically updated by Azure
   - Reduces maintenance burden

3. **Organize Rules by Tier**
   - Web tier rules (80, 443)
   - App tier rules (custom ports)
   - Database tier rules (1433, 5432)

4. **Priority Planning**
   - Leave gaps (100, 200, 300) for future rules
   - Document rule purposes

5. **Subnet-Level vs NIC-Level**
   - Use subnet NSGs for consistent tier policies
   - Use NIC NSGs only for exceptions

6. **Regular Audits**
   - Review unused rules quarterly
   - Document rule justifications
   - Update for compliance changes

---

## Application Security Groups (ASGs)

### ASG Overview

Application Security Groups enable you to group VMs logically and define network security policies based on application function rather than explicit IP addresses.

**Benefits:**
- Simplifies rule management at scale
- Reduces IP address dependencies
- Enables dynamic VM grouping
- Scales with application changes

**Use Cases:**
- Multi-tier applications (web, app, database tiers)
- Microservices architecture
- Segregating environments (prod, staging, dev)
- Temporary groupings for specific projects

### Creating ASGs

**PowerShell:**

```powershell
# Create ASGs
$webAsg = New-AzApplicationSecurityGroup `
  -ResourceGroupName "myRG" `
  -Name "asg-web" `
  -Location "eastus"

$dbAsg = New-AzApplicationSecurityGroup `
  -ResourceGroupName "myRG" `
  -Name "asg-database" `
  -Location "eastus"

# Associate VMs to ASG
$nic = Get-AzNetworkInterface -Name "webvm1-nic" -ResourceGroupName "myRG"
$nic.ApplicationSecurityGroups = @($webAsg)
$nic | Set-AzNetworkInterface
```

**Azure CLI:**

```bash
# Create ASG
az network asg create \
  --resource-group myRG \
  --name asg-web

# Associate to NIC
az network nic update \
  --resource-group myRG \
  --name webvm1-nic \
  --application-security-groups asg-web
```

### Using ASGs in Security Rules

**Example: Allow web traffic to web ASG**

```powershell
# Create rule using ASG as destination
$rule = New-AzNetworkSecurityRuleConfig `
  -Name "AllowInternetToWeb" `
  -Protocol Tcp `
  -Direction Inbound `
  -Priority 100 `
  -SourceAddressPrefix Internet `
  -SourcePortRange * `
  -DestinationApplicationSecurityGroup $webAsg `
  -DestinationPortRange 80,443 `
  -Access Allow

# Add rule to NSG
$nsg = Get-AzNetworkSecurityGroup -ResourceGroupName "myRG" -Name "myNSG"
$nsg.SecurityRules += $rule
$nsg | Set-AzNetworkSecurityGroup
```

### ASG Scenarios

**Three-Tier Application:**

```
Internet (source)
  ↓ (Port 80, 443)
[asg-web] (web VMs)
  ↓ (Port 8080)
[asg-api] (app VMs)
  ↓ (Port 3306)
[asg-database] (DB VMs)
```

**Rules Implementation:**

```powershell
# Allow Internet to Web
New-AzNetworkSecurityRuleConfig -Name "AllowIntToWeb" `
  -SourceAddressPrefix Internet `
  -DestinationApplicationSecurityGroup $webAsg `
  -DestinationPortRange 80,443 `
  -Protocol Tcp -Direction Inbound -Priority 100 `
  -Access Allow

# Allow Web to API
New-AzNetworkSecurityRuleConfig -Name "AllowWebToAPI" `
  -SourceApplicationSecurityGroup $webAsg `
  -DestinationApplicationSecurityGroup $apiAsg `
  -DestinationPortRange 8080 `
  -Protocol Tcp -Direction Inbound -Priority 110 `
  -Access Allow

# Allow API to Database
New-AzNetworkSecurityRuleConfig -Name "AllowAPIToDb" `
  -SourceApplicationSecurityGroup $apiAsg `
  -DestinationApplicationSecurityGroup $dbAsg `
  -DestinationPortRange 3306 `
  -Protocol Tcp -Direction Inbound -Priority 120 `
  -Access Allow
```

---

## Azure Bastion

### Bastion Overview

Azure Bastion is a fully managed PaaS service providing secure RDP/SSH connectivity to VMs without exposing public IP addresses.

**Key Benefits:**
- No public IP required on VMs
- Secure RDP/SSH over TLS (port 443)
- No additional software or agents needed
- Portal-based or native client access
- Protection against port scanning
- Session recording (Premium SKU)
- Microsoft Entra ID authentication support

**When to Use:**
- Secure VM access without public IPs
- Administrative management of VMs
- Jump box alternative
- Compliance requirements (no public exposure)
- Reduced attack surface

### Bastion Architecture

**Components:**
1. Bastion subnet (`AzureBastionSubnet`, /26 minimum)
2. Bastion host (Standard or Premium SKU)
3. Public IP (for Bastion host only)
4. VMs without public IPs

**Traffic Flow:**
```
User (Azure Portal) 
  → HTTPS (port 443) 
  → Bastion Host 
  → Private IP RDP/SSH 
  → VM (no public IP)
```

### Deploying Azure Bastion

**PowerShell:**

```powershell
# Create Bastion subnet
$bastSubnet = New-AzVirtualNetworkSubnetConfig `
  -Name "AzureBastionSubnet" `
  -AddressPrefix "10.0.255.0/26"

# Create VNet with Bastion subnet
$vnet = New-AzVirtualNetwork `
  -ResourceGroupName "myRG" `
  -Location "eastus" `
  -Name "myVNet" `
  -AddressPrefix "10.0.0.0/16" `
  -Subnet $bastSubnet

# Create public IP for Bastion
$pip = New-AzPublicIpAddress `
  -ResourceGroupName "myRG" `
  -Location "eastus" `
  -Name "bastion-pip" `
  -AllocationMethod Static `
  -Sku Standard

# Create Bastion host
New-AzBastion `
  -ResourceGroupName "myRG" `
  -Name "myBastion" `
  -PublicIpAddress $pip `
  -VirtualNetwork $vnet `
  -Sku Standard
```

**Azure CLI:**

```bash
# Create Bastion
az network bastion create \
  --resource-group myRG \
  --name myBastion \
  --vnet-name myVNet \
  --public-ip-address bastion-pip \
  --location eastus
```

### Bastion SKUs

| Feature | Standard | Premium |
|---------|----------|---------|
| **RDP/SSH** | Yes | Yes |
| **Session Recording** | No | Yes |
| **Native Client Support** | No | Yes |
| **Entra ID Auth** | No | Yes (Preview) |
| **Max Concurrent Sessions** | 25 | Varies |
| **Cost** | Lower | Higher |

### NSG Configuration for Bastion

**Required Bastion Subnet NSG Rules:**

| Direction | Source | Port | Protocol | Purpose |
|-----------|--------|------|----------|---------|
| **Inbound** | Internet | 443 | TCP | HTTPS for Bastion |
| **Inbound** | GatewayManager | 443 | TCP | Azure platform management |
| **Outbound** | Any | 443 | TCP | Outbound HTTPS |
| **Outbound** | Any | 3389 | TCP | RDP to VMs |
| **Outbound** | Any | 22 | TCP | SSH to VMs |

**VM Subnet NSG Rules:**

```powershell
# Allow Bastion inbound to VMs
$rule = New-AzNetworkSecurityRuleConfig `
  -Name "AllowBastionInbound" `
  -Protocol * `
  -Direction Inbound `
  -Priority 100 `
  -SourceAddressPrefix "10.0.255.0/26" `
  -SourcePortRange * `
  -DestinationAddressPrefix * `
  -DestinationPortRange 22,3389 `
  -Access Allow
```

### Connecting via Bastion

**Portal Method:**
1. Navigate to VM in Azure Portal
2. Click **Connect** → **Bastion**
3. Enter username and password (or SSH key)
4. Click **Connect**

**Native Client Method (Premium SKU):**
```bash
# RDP via native client
az network bastion rdp \
  --name myBastion \
  --resource-group myRG \
  --target-resource-id /subscriptions/.../vm1

# SSH via native client
az network bastion ssh \
  --name myBastion \
  --resource-group myRG \
  --target-resource-id /subscriptions/.../vm1
```

### Bastion Best Practices

1. **Use Dedicated Subnet**
   - Separate `/26` subnet for Bastion only
   - Easier to manage and scale

2. **NSG Configuration**
   - Restrictive inbound rules
   - Allow Bastion traffic only

3. **Regional Deployment**
   - Deploy in each region with VMs
   - Reduces latency

4. **Monitor Access**
   - Enable logging for audit trail
   - Review session recordings (Premium)

5. **Enforce Entra ID**
   - Use Entra ID authentication when available
   - Requires Premium SKU

---

## Private Endpoints and Service Endpoints

### Service Endpoints

Service endpoints provide secure, direct connectivity to Azure services over the Azure backbone network.

**Characteristics:**
- No public internet traversal
- Works at subnet level
- VNet private IP appears as source
- Supported services: Storage, SQL, Cosmos DB, Event Hubs, Service Bus, Key Vault

**Configuration:**

```powershell
# Add service endpoint to subnet
$vnet = Get-AzVirtualNetwork -ResourceGroupName "myRG" -Name "myVNet"
$subnet = Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name "mySubnet"
$subnet.ServiceEndpoints.Add((New-AzServiceEndpointPolicy -ServiceEndpointPolicyDefinition @{
    Service = 'Microsoft.Storage'
}))
Set-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet @subnet
$vnet | Set-AzVirtualNetwork
```

**Scope:** Entire Azure service (all Storage accounts, all SQL servers)

### Private Endpoints

Private endpoints provide instance-level private connectivity using private IPs.

**Characteristics:**
- Creates NIC with private IP in VNet
- Granular access (specific resource instance)
- Works with on-premises (via ExpressRoute/VPN)
- Requires DNS configuration
- Prevents data exfiltration

**Configuration:**

```powershell
# Create private endpoint
$privateEndpoint = New-AzPrivateEndpoint `
  -ResourceGroupName "myRG" `
  -Name "storage-pe" `
  -VirtualNetwork $vnet `
  -Subnet $subnet `
  -PrivateLinkServiceConnection @{
    Name = "storage-connection"
    PrivateLinkServiceId = "/subscriptions/.../providers/Microsoft.Storage/storageAccounts/mystg"
    GroupId = "blob"
  }

# Link to private DNS zone
$dnsZone = Get-AzPrivateDnsZone -ResourceGroupName "myRG" -Name "privatelink.blob.core.windows.net"
New-AzPrivateDnsZoneGroup `
  -ResourceGroupName "myRG" `
  -PrivateDnsZoneName "privatelink.blob.core.windows.net" `
  -Name "storage-dnsgroup" `
  -PrivateEndpoint $privateEndpoint
```

### Service Endpoint vs Private Endpoint

| Aspect | Service Endpoint | Private Endpoint |
|--------|------------------|------------------|
| **Scope** | Entire service | Individual resource |
| **Source Visibility** | Private IP | Private IP |
| **On-Premises Access** | No | Yes (via VPN/ExpressRoute) |
| **Data Exfiltration Protection** | No | Yes |
| **DNS Changes** | No | Yes |
| **Setup Complexity** | Simple | Moderate |
| **Cost** | No additional cost | Hourly + data transfer |
| **Multi-Region** | Supported | Supported |

### Recommended Approach

**Use Service Endpoints when:**
- All traffic from specific subnets
- Cost optimization important
- Simple setup sufficient
- No on-premises access needed

**Use Private Endpoints when:**
- Fine-grained access control needed
- On-premises connectivity required
- Data exfiltration prevention critical
- Multi-tenant scenarios

---

## PowerShell & Azure CLI Commands Reference

### NSG Management

```powershell
# Create NSG
New-AzNetworkSecurityGroup -ResourceGroupName "myRG" -Location "eastus" -Name "myNSG"

# Get NSG
Get-AzNetworkSecurityGroup -ResourceGroupName "myRG" -Name "myNSG"

# Add rule
Add-AzNetworkSecurityRuleConfig -Name "AllowHTTP" `
  -Protocol Tcp -Direction Inbound -Priority 100 `
  -SourceAddressPrefix Internet -SourcePortRange * `
  -DestinationAddressPrefix * -DestinationPortRange 80 `
  -Access Allow -NetworkSecurityGroup $nsg

# Remove rule
Remove-AzNetworkSecurityRuleConfig -Name "AllowHTTP" -NetworkSecurityGroup $nsg

# Associate to subnet
Set-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet `
  -Name "mySubnet" -NetworkSecurityGroup $nsg

# Delete NSG
Remove-AzNetworkSecurityGroup -ResourceGroupName "myRG" -Name "myNSG" -Force
```

### ASG Management

```powershell
# Create ASG
New-AzApplicationSecurityGroup -ResourceGroupName "myRG" -Name "asg-web" -Location "eastus"

# Get ASG
Get-AzApplicationSecurityGroup -ResourceGroupName "myRG" -Name "asg-web"

# Associate to NIC
$nic = Get-AzNetworkInterface -ResourceGroupName "myRG" -Name "vm1-nic"
$nic.ApplicationSecurityGroups = @($asg)
$nic | Set-AzNetworkInterface

# Delete ASG
Remove-AzApplicationSecurityGroup -ResourceGroupName "myRG" -Name "asg-web" -Force
```

### Bastion Management

```powershell
# Create Bastion
New-AzBastion -ResourceGroupName "myRG" -Name "myBastion" `
  -VirtualNetwork $vnet -PublicIpAddress $pip

# Get Bastion
Get-AzBastion -ResourceGroupName "myRG" -Name "myBastion"

# Delete Bastion
Remove-AzBastion -ResourceGroupName "myRG" -Name "myBastion" -Force
```

---

## Exam Tips

### Key Concepts

1. **NSG Rule Priority:** Lower number = higher priority (100 before 200)
2. **Stateful Filtering:** Return traffic automatically allowed
3. **Default Deny Inbound:** Must explicitly allow required ports
4. **Subnet vs NIC NSG:** Both apply; combined rules determine access
5. **Service Tags:** Use for Azure services instead of IP ranges
6. **ASGs:** Enable dynamic grouping without IP dependencies
7. **Bastion:** No public IPs needed for VMs
8. **Service Endpoints:** VNet-level private connectivity
9. **Private Endpoints:** Resource-level private connectivity
10. **Deny by Default:** Security best practice

### Common Scenarios

| Scenario | Solution |
|----------|----------|
| **Allow web traffic** | NSG rule: Inbound, Port 80/443, Priority 100 |
| **Restrict database access** | ASG for database VMs, explicit source rules |
| **Secure VM management** | Azure Bastion, no public IP on VM |
| **Hybrid cloud access** | Private endpoint + VPN/ExpressRoute |
| **Compliance: no public exposure** | Bastion for access, private endpoints for services |
| **Multi-tier app security** | ASGs for each tier, rules between tiers |

---

## Exam Scenarios

### Scenario 1: Multi-Tier Web Application Security

**Requirements:**
- Web tier (front-end)
- App tier (middle tier)
- Database tier (backend)
- Internet access to web tier only
- No direct internet access to app/db tiers
- Admin access via Azure Bastion

**Configuration:**

1. **Create ASGs:**
   - asg-web (web tier VMs)
   - asg-api (app tier VMs)
   - asg-database (database tier VMs)

2. **Create NSG with rules:**
   ```
   Priority 100: Allow Internet → asg-web (Port 80, 443)
   Priority 110: Allow asg-web → asg-api (Port 8080)
   Priority 120: Allow asg-api → asg-database (Port 3306)
   Priority 200: Deny all other inbound
   ```

3. **Deploy Bastion:**
   - Bastion subnet (10.0.255.0/26)
   - Bastion host
   - NSG rule: Bastion → asg-web/api/db (Port 22, 3389)

4. **NSG on database subnet:**
   - Allow asg-api inbound (port 3306)
   - Deny internet inbound
   - Deny app-tier outbound to internet (except DNS)

**Validation:**
- User connects via Bastion to web VM
- Web VM reaches app tier (port 8080)
- App tier reaches database (port 3306)
- Direct internet access blocked for app/db tiers

### Scenario 2: Hybrid Cloud with Private Endpoints

**Requirements:**
- On-premises network (ExpressRoute)
- Azure storage account (private access only)
- VMs access storage via private endpoint
- On-premises systems also access storage

**Configuration:**

1. **Create Private Endpoint:**
   - Storage account private endpoint
   - Linked to application subnet
   - DNS integrated with private zone

2. **Private DNS Zone:**
   - Zone: privatelink.blob.core.windows.net
   - Record: storage account CNAME → private IP
   - Link to VNet + on-premises DNS forwarder

3. **NSG Configuration:**
   - Allow VM outbound to private endpoint IP (port 443)
   - Firewall rule on storage: Allow private endpoint only

4. **On-Premises Connection:**
   - DNS forwarder routes privatelink.blob to private endpoint
   - ExpressRoute provides connectivity
   - On-premises systems access via private endpoint

**Validation:**
- VMs reach storage via private IP (10.0.x.x)
- On-premises systems reach storage via private IP
- No public IP access allowed
- Compliance: All traffic on Azure backbone

### Scenario 3: Secure Administrative Access

**Requirements:**
- VMs have no public IPs
- Administrators manage from home/office
- No VPN required
- Session audit trail
- MFA enforcement

**Configuration:**

1. **Azure Bastion (Premium SKU):**
   - Deployed in Bastion subnet
   - Static public IP for Bastion only
   - VMs: Private IPs only

2. **Authentication:**
   - Entra ID integration (Premium SKU)
   - Conditional access policies
   - MFA required for Bastion connection

3. **NSG for Bastion:**
   - Inbound: Internet → port 443 (HTTPS)
   - Outbound: Port 22, 3389 to VM subnets
   - VM NSG: Bastion IP range → VM (port 22, 3389)

4. **Monitoring:**
   - Enable session recording (Premium)
   - Azure Monitor for Bastion logs
   - Alert on failed connection attempts

5. **Access Flow:**
   - Admin authenticates to Azure (MFA)
   - Selects VM from portal
   - Bastion establishes session
   - Session recorded automatically

**Validation:**
- No public IPs on VMs
- Access only through Bastion
- All sessions audited
- MFA prevents unauthorized access

---

## Troubleshooting Network Security

### VMs Cannot Communicate

**Troubleshooting Steps:**

1. **Check NSG Rules:**
```powershell
# View effective rules on NIC
Get-AzEffectiveNetworkSecurityGroup -NetworkInterfaceName "vm1-nic" -ResourceGroupName "myRG"

# Check specific rule
$nsg = Get-AzNetworkSecurityGroup -ResourceGroupName "myRG" -Name "myNSG"
$nsg.SecurityRules | Where-Object {$_.Name -eq "AllowHTTP"}
```

2. **Verify ASG Membership:**
```powershell
# Check if NIC is in correct ASG
$nic = Get-AzNetworkInterface -ResourceGroupName "myRG" -Name "vm1-nic"
$nic.ApplicationSecurityGroups
```

3. **Test Connectivity:**
```powershell
# Network Watcher connection troubleshoot
Test-AzNetworkWatcherConnectivity -TargetResourceId $vm2.Id -SourceResourceId $vm1.Id
```

### Cannot Access via Bastion

**Causes and Solutions:**

| Issue | Cause | Solution |
|-------|-------|----------|
| Connection timeout | NSG blocking 22/3389 | Add NSG rule allowing Bastion IP range |
| Connection refused | VM not running agent | No agent needed; verify VM is running |
| DNS resolution fails | Private DNS not configured | Link Bastion subnet to DNS zone |
| Session recording fails (Premium) | Storage account access | Verify Bastion has permission to storage |

### Private Endpoint DNS Issues

**Troubleshooting:**

```powershell
# Verify DNS resolution
Resolve-DnsName "mystg.blob.core.windows.net"
# Should resolve to private IP (10.x.x.x)

# Check private DNS zone configuration
Get-AzPrivateDnsZone -ResourceGroupName "myRG"
Get-AzPrivateDnsRecordSet -ResourceGroupName "myRG" -ZoneName "privatelink.blob.core.windows.net"

# Verify VNet link
Get-AzPrivateDnsVirtualNetworkLink -ResourceGroupName "myRG" -ZoneName "privatelink.blob.core.windows.net"
```

### Service Endpoint Not Working

**Verification:**

```powershell
# Confirm service endpoint on subnet
$subnet = Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name "mySubnet"
$subnet.ServiceEndpoints

# Check storage firewall rules
$sa = Get-AzStorageAccount -ResourceGroupName "myRG" -Name "mystg"
$sa.NetworkRuleSet.VirtualNetworkRules
```

---

## Performance and Best Practices

### NSG Optimization

- **Minimize Rule Count:** Consolidate similar rules
- **Priority Gaps:** Use 100, 200, 300 for future additions
- **Service Tags:** Use over IP ranges when possible
- **Regular Audits:** Remove unused rules quarterly
- **Rule Documentation:** Include business justification

### ASG Best Practices

- **Logical Grouping:** Based on application function
- **Naming Convention:** Prefix with tier (asg-web, asg-api)
- **Automation:** Use IaC (ARM, Terraform) for consistency
- **Scale Considerations:** ASGs scale better than IP-based rules

### Bastion Best Practices

- **Regional Deployment:** One per region for reduced latency
- **Premium for Production:** Enables session recording and Entra ID
- **Monitoring:** Enable diagnostic logs and activity monitoring
- **Cost Optimization:** Use Standard SKU for non-critical workloads

### Private Endpoint Best Practices

- **Private DNS Zones:** Always use for consistent name resolution
- **Dedicated Endpoint:** One per critical service instance
- **Approval Workflow:** Review and manage endpoint connections
- **Monitoring:** Log endpoint access for audit compliance

