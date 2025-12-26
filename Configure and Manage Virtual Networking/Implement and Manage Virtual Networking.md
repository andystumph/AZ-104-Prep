# Implement and Manage Virtual Networking

## Azure Virtual Network Overview

Azure Virtual Networks (VNets) are the foundation of networking in Azure. They provide an isolated, private network space where you can deploy Azure resources and control communication between them.

### Key Concepts

**Virtual Network (VNet)**
- Logical boundary and private IP address space in Azure
- Scoped to a single region and subscription
- Resources deployed in a VNet communicate using private IP addresses
- Pre-configured with 10.0.0.0/16 by default (customizable)
- Can be connected to other VNets via peering or gateways

**Subnets**
- Subdivisions of a VNet's address space
- Enable segmentation and security at network level
- Resources in same VNet but different subnets can communicate without configuration
- First four IP addresses reserved in each subnet:
  - .0: Network address
  - .1: Gateway address (reserved for default gateway)
  - .2-3: DNS addresses
  - Example: 10.0.0.0/24 reserves 10.0.0.0, .1, .2, .3

**Address Space Planning**
- Use RFC 1918 private address ranges only:
  - 10.0.0.0/8 (10.0.0.0 - 10.255.255.255)
  - 172.16.0.0/12 (172.16.0.0 - 172.31.255.255)
  - 192.168.0.0/16 (192.168.0.0 - 192.168.255.255)
- Plan for non-overlapping spaces across regions and on-premises networks
- CIDR notation: /16 = 65,536 addresses, /24 = 256 addresses, /28 = 16 addresses
- Azure reserves first 4 addresses per subnet
- Available usable addresses = subnet addresses - 5

**Reserved Address Ranges (cannot use)**
- 224.0.0.0/4 (multicast)
- 255.255.255.255/32 (broadcast)
- 127.0.0.0/8 (loopback)
- 169.254.0.0/16 (link-local)
- 168.63.129.16/32 (internal DNS)

### VNet Scope and Limits

| Aspect | Details |
|--------|---------|
| **Region** | Single region per VNet; use peering for multi-region |
| **Subscription** | Single subscription per VNet |
| **Max subnets** | Unlimited (limited by CIDR/address space) |
| **Max VNets per subscription** | 50 (default, can increase via support) |
| **Max peered VNets** | 500 (local region); 1000 with Azure Virtual Network Manager |
| **Multiple address prefixes** | Yes, can add/remove (with potential downtime for peered VNets) |

---

## IP Addressing

### Public IP Addresses

**Purpose**
- Enable inbound/outbound communication with internet and public Azure services
- Not part of VNet's private address space
- Assigned to NICs, Load Balancers, VPN Gateways, NAT Gateways
- Allocated from Azure's public IP pool

**Allocation Methods**

| Method | Cost | Assignment | Lifecycle |
|--------|------|-----------|-----------|
| Static | Yes | At resource creation; doesn't change | Persists until resource deleted |
| Dynamic | No | From Azure pool; changes on stop/deallocate | Released when resource deallocated |

**SKU Levels**

| SKU | Features | Use Case |
|-----|----------|----------|
| Basic | Standard routing, no zones | Compatibility, test environments |
| Standard | Zone support, guaranteed quality, DDoS protection | Production workloads, HA requirements |

**Static Outbound IP Address**
- Use NAT Gateway (recommended) to assign static public IP
- Associate with subnet; all outbound traffic uses NAT gateway's IP
- Supports up to 16 public IP addresses or prefixes per NAT gateway
- Alternative: Attach public IP directly to VM NIC (1:1 mapping)

### Private IP Addresses

**Allocation Methods**

| Method | Assignment | Usefulness |
|--------|-----------|-----------|
| Dynamic | DHCP from subnet's range | Temporary, non-critical workloads |
| Static | Manual assignment from subnet range | Critical services, DNS records, application requirements |

**DNS Name Resolution**
- Private FQDN: `[vmname].internal.cloudapp.net` (Azure-provided)
- Custom domains use Azure Private DNS Zones (recommended)
- Automatic registration available if autoregistration enabled on zone link
- Can configure custom DNS servers for on-premises name resolution

### IPv4 and IPv6 Support

- VNets support IPv4-only (default) or dual-stack (IPv4 + IPv6)
- IPv6 subnets must be exactly /64 in size
- IPv6 preferred for address exhaustion scenarios
- Not all services support IPv6; verify before deploying

---

## Creating Virtual Networks

### Azure Portal Method

**Steps:**
1. Navigate to Virtual networks > + Create
2. Enter basic info (subscription, resource group, VNet name, region)
3. Define address space (e.g., 10.0.0.0/16)
4. Create subnets with appropriate CIDR ranges
5. Configure DNS settings (use Azure default or custom servers)
6. Add tags for organization
7. Review and create

### PowerShell Method

```powershell
# Create subnet configuration
$subnetConfig = New-AzVirtualNetworkSubnetConfig `
  -Name 'mySubnet' `
  -AddressPrefix '192.168.1.0/24'

# Create virtual network
$vnet = New-AzVirtualNetwork `
  -ResourceGroupName 'myResourceGroup' `
  -Location 'eastus' `
  -Name 'myVNet' `
  -AddressPrefix '192.168.0.0/16' `
  -Subnet $subnetConfig

# Add additional subnet to existing VNet
$vnet = Get-AzVirtualNetwork -ResourceGroupName 'myResourceGroup' -Name 'myVNet'
Add-AzVirtualNetworkSubnetConfig -Name 'mySubnet2' `
  -AddressPrefix '192.168.2.0/24' `
  -VirtualNetwork $vnet
$vnet | Set-AzVirtualNetwork
```

### Azure CLI Method

```bash
# Create VNet with subnet
az network vnet create \
  --resource-group myResourceGroup \
  --name myVNet \
  --address-prefix 10.0.0.0/16 \
  --subnet-name mySubnet \
  --subnet-prefix 10.0.0.0/24

# Add subnet to existing VNet
az network vnet subnet create \
  --resource-group myResourceGroup \
  --vnet-name myVNet \
  --name mySubnet2 \
  --address-prefix 10.0.1.0/24
```

### Expanding Address Space

**Adding prefix to existing VNet:**
- Can be done without downtime
- Peered VNets require resync operation after adding space
- New instances can use new address space immediately

```powershell
# Add address prefix
$vnet = Get-AzVirtualNetwork -ResourceGroupName 'myRG' -Name 'myVNet'
$vnet.AddressSpace.AddressPrefixes.Add('10.1.0.0/16')
$vnet | Set-AzVirtualNetwork

# Resync peering after address space change
$peeringName = 'vnet1-to-vnet2'
Get-AzVirtualNetworkPeering -VirtualNetworkName 'vnet1' -ResourceGroupName 'myRG' `
  -Name $peeringName | Update-AzVirtualNetworkPeering
```

---

## Virtual Network Peering

### Overview

Virtual Network Peering connects two or more VNets, enabling resources to communicate over private IP addresses. Traffic uses Microsoft's backbone network, not the public internet.

### Types of Peering

**Regional VNet Peering (Local)**
- Connects VNets in same region
- Low latency, high bandwidth
- Minimal cost
- Full connectivity between resources

**Global VNet Peering**
- Connects VNets across regions
- Slightly higher latency than local peering
- Charges apply for cross-region traffic
- Useful for multi-region deployments

### Key Benefits

- Seamless connectivity with no downtime
- Private communication (not over internet)
- Works across subscriptions, Azure tenants, and regions (with limits)
- Transit peering with hub-and-spoke topology
- Data transfer costs only for cross-region peering

### Peering Constraints

| Constraint | Limit | Notes |
|-----------|-------|-------|
| **VNets per peering** | 500 (1000 with Virtual Network Manager) | Default limit; requires support request to increase |
| **Basic Load Balancer** | Not supported over global peering | Must use Standard Load Balancer |
| **Resource access** | Some resources unsupported over global peering | VMs behind Basic ILB, Redis (Basic), App Gateway v1, Scale Sets (Basic), Service Fabric |

**Global Peering Unsupported Resources:**
- VMs behind Basic Internal Load Balancer
- Azure Cache for Redis (Basic tier)
- Azure Application Gateway v1
- Virtual Machine Scale Sets (Basic)
- Azure Service Fabric
- Azure API Management (stv1)
- Microsoft Entra Domain Services
- Azure Logic Apps
- Azure HDInsight
- Azure Batch
- App Service Environment v1 and v2

### Creating Peering

**PowerShell:**

```powershell
# Get VNets
$vnet1 = Get-AzVirtualNetwork -ResourceGroupName 'test-rg' -Name 'vnet-1'
$vnet2 = Get-AzVirtualNetwork -ResourceGroupName 'test-rg-2' -Name 'vnet-2'

# Create peering from vnet1 to vnet2
Add-AzVirtualNetworkPeering -Name 'vnet1-to-vnet2' `
  -VirtualNetwork $vnet1 `
  -RemoteVirtualNetworkId $vnet2.Id

# Create peering from vnet2 to vnet1 (bidirectional)
Add-AzVirtualNetworkPeering -Name 'vnet2-to-vnet1' `
  -VirtualNetwork $vnet2 `
  -RemoteVirtualNetworkId $vnet1.Id

# Configure peering settings (allow forwarded traffic, allow gateway transit)
$peering = Get-AzVirtualNetworkPeering -VirtualNetworkName 'vnet1' `
  -ResourceGroupName 'test-rg' -Name 'vnet1-to-vnet2'
$peering.AllowForwardedTraffic = $true
$peering.AllowGatewayTransit = $true
$peering | Set-AzVirtualNetworkPeering
```

**Azure CLI:**

```bash
# Create peering
az network vnet peering create \
  --resource-group test-rg \
  --vnet-name vnet-1 \
  --name vnet1-to-vnet2 \
  --remote-vnet '/subscriptions/{subscription-id}/resourceGroups/test-rg-2/providers/Microsoft.Network/virtualNetworks/vnet-2' \
  --allow-vnet-access true
```

### Peering Configuration Options

| Setting | Purpose | Impact |
|---------|---------|--------|
| **Allow virtual network access** | Enable communication between VNets | Required for peering to work |
| **Allow forwarded traffic** | Allow traffic from non-peered sources | Enables transit routing |
| **Allow gateway transit** | Use peer's VPN/ExpressRoute gateway | Reduces cost; centralizes gateway |
| **Use remote gateway** | Route through peer's gateway | For on-prem connectivity via peer's gateway |

### Hub-and-Spoke Topology

**Architecture:**
- Central hub VNet (gateway, firewall, shared services)
- Spoke VNets connected to hub
- Spokes communicate via hub or directly (with transit peering)

**Advantages:**
- Simplified management
- Reduced gateway costs
- Centralized security controls
- Scalable design

**Configuration:**
1. Create hub and spoke VNets
2. Peer spokes to hub
3. Enable "Allow forwarded traffic" on all peers
4. Configure route tables in spokes with NVA as next hop (for inspection)
5. Enable "Use hub as gateway" for on-prem connectivity

### Service Chaining

Route traffic through Network Virtual Appliances (NVA) for inspection, logging, or policy enforcement using User-Defined Routes (UDRs).

**Steps:**
1. Deploy NVA (firewall, proxy) in hub VNet
2. Create route table with routes pointing to NVA's IP
3. Associate route table to spoke subnets
4. Enable forwarded traffic on hub peering
5. Configure NVA to forward traffic

---

## User-Defined Routes (UDRs)

### Overview

User-Defined Routes (custom routes) allow you to override Azure's default routing and direct traffic to specific next hops.

### Route Table Basics

- Associate route table to zero or more subnets
- Each subnet can have one route table
- Routes in table combined with default system routes
- UDRs override default routes if address prefix matches
- Azure uses longest prefix match for routing decision

**Route Priority (longest prefix match):**
1. Most specific prefix
2. If equal, Azure selects first in evaluation order

**Route Table Limits:**
- Default: 400 routes per table
- With Azure Virtual Network Manager: 1000 routes per table

### Creating Route Tables

**PowerShell:**

```powershell
# Create route table
$routeTable = New-AzRouteTable `
  -ResourceGroupName 'myResourceGroup' `
  -Location 'eastus' `
  -Name 'myRouteTable'

# Add custom route (traffic to 0.0.0.0/0 goes to firewall)
$route = @{
    Name = 'ToFirewall'
    RouteTable = $routeTable
    AddressPrefix = '0.0.0.0/0'
    NextHopType = 'VirtualAppliance'
    NextHopIpAddress = '10.0.1.4' # Firewall IP
}
Add-AzRouteConfig @route
$routeTable | Set-AzRouteTable

# Associate route table to subnet
$vnet = Get-AzVirtualNetwork -ResourceGroupName 'myResourceGroup' -Name 'myVNet'
$subnet = Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name 'mySubnet'
$subnet.RouteTable = $routeTable
Set-AzVirtualNetworkSubnetConfig @subnet
$vnet | Set-AzVirtualNetwork
```

**Azure CLI:**

```bash
# Create route table
az network route-table create \
  --resource-group myResourceGroup \
  --name myRouteTable

# Add route
az network route-table route create \
  --resource-group myResourceGroup \
  --route-table-name myRouteTable \
  --name ToFirewall \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.0.1.4

# Associate to subnet
az network vnet subnet update \
  --resource-group myResourceGroup \
  --vnet-name myVNet \
  --name mySubnet \
  --route-table myRouteTable
```

### Next Hop Types

| Type | Usage | Example |
|------|-------|---------|
| **Virtual Appliance** | Route to firewall, proxy, or NVA | 10.0.1.4 (NVA private IP) |
| **Virtual Network Gateway** | Route to VPN gateway (for BGP routes) | VPN gateway in VNet |
| **Virtual Network** | Override default routing within VNet | Force traffic through different path |
| **Internet** | Explicit routing to internet | 0.0.0.0/0 to internet |
| **None** | Drop traffic (deny) | 192.168.1.0/24 with next hop None |
| **VirtualNetworkServiceEndpoint** | Route to service endpoint | Automatic (not manual) |

### 0.0.0.0/0 Address Prefix

**Purpose:** Force all internet-bound traffic through specific next hop (firewall, proxy, NAT gateway).

**Considerations:**
- Takes precedence over default internet routing
- Requires destination to have internet access (firewall must be on internet path)
- Without proper configuration, creates disconnected network
- Common use: force all traffic through firewall for inspection

**Configuration Example:**
```powershell
# Force all traffic to firewall
$route = @{
    Name = 'DefaultRoute'
    RouteTable = $routeTable
    AddressPrefix = '0.0.0.0/0'
    NextHopType = 'VirtualAppliance'
    NextHopIpAddress = '10.0.1.4' # Firewall IP
}
Add-AzRouteConfig @route
```

### Viewing Routes

**Effective Routes for VM:**
```powershell
# View effective routes on NIC
Get-AzEffectiveRouteTable -ResourceGroupName 'myRG' -NetworkInterfaceName 'myVM-nic'

# Azure Portal: VM > Networking > Network Interface > Effective Routes
```

Routes shown include:
- System routes
- Propagated BGP routes
- UDRs from associated route table
- Service endpoint routes

---

## DNS and Name Resolution

### Azure-Provided DNS

**Default behavior:**
- VMs automatically receive Azure DNS servers (168.63.129.16)
- Internal hostnames resolvable within VNet
- FQDN format: `[vmname].internal.cloudapp.net`
- No configuration required
- Automatically handles DHCP assignments

**Limitations:**
- Only internal to VNet
- No custom zone management
- Different DNS suffix per classic cloud service (if using classic model)

### Azure Private DNS Zones

**Purpose:** Manage private DNS records for VNet-integrated resources; preferred for name resolution.

**Benefits:**
- Custom domain names for private resources
- Automatic VM registration (with autoregistration enabled)
- Split-horizon DNS capability
- Supports multiple VNet links

**Configuration:**
```powershell
# Create private DNS zone
$dnsZone = New-AzPrivateDnsZone -Name 'contoso.com' `
  -ResourceGroupName 'myResourceGroup'

# Link to VNet with autoregistration
$vnet = Get-AzVirtualNetwork -ResourceGroupName 'myResourceGroup' -Name 'myVNet'
$link = New-AzPrivateDnsVirtualNetworkLink `
  -ZoneName 'contoso.com' `
  -ResourceGroupName 'myResourceGroup' `
  -Name 'myVNet-link' `
  -VirtualNetworkId $vnet.Id `
  -EnableRegistration
```

### Custom DNS Servers

**When to use:**
- On-premises DNS integration
- Active Directory domain controllers
- Conditional forwarding to external DNS
- Specific organizational requirements

**Configuration:**
```powershell
# Set custom DNS servers
$dnsServers = @('10.0.1.4', '10.0.1.5')
$vnet = Get-AzVirtualNetwork -ResourceGroupName 'myRG' -Name 'myVNet'
$vnet.DhcpOptions.DnsServers = $dnsServers
$vnet | Set-AzVirtualNetwork
```

**Custom Server Requirements:**
- Must be accessible from VNet (TCP/UDP port 53)
- Should forward queries to Azure (168.63.129.16) for external resolution
- Forwarding timeout > 4 seconds (to prevent Private DNS zone issues)
- Should be highly available (multiple servers recommended)
- Disable DNS record scavenging (Azure uses long DHCP leases)

### Azure DNS Private Resolver

**Purpose:** Hybrid DNS resolution between Azure and on-premises without managing DNS servers.

**Benefits:**
- No VM management required
- High availability built-in
- Automatic scaling
- Conditional forwarding rules
- Fully managed PaaS service

**Use cases:**
- On-prem DNS integration at scale
- VNet-to-VNet DNS resolution
- Conditional forwarding without NVA

---

## Network Outbound Connectivity

### Default Outbound Access (Deprecated)

**Current status:** Being phased out; explicitly configure outbound connectivity.

**Timeline:** March 31, 2026 - new VNets will default to private subnets (no default outbound access).

**Current behavior:** VMs without explicit outbound method assigned a default Microsoft-owned public IP (not recommended for production).

### NAT Gateway

**Purpose:** Provide static, managed outbound internet connectivity for VNet subnets.

**Advantages:**
- Static public IP for outbound traffic (all VMs in subnet use same IP)
- No SNAT port exhaustion (highly scalable)
- Managed by Azure (no configuration needed)
- Works with Standard SKU resources only
- Takes precedence over other outbound methods

**Configuration:**

```powershell
# Create public IP for NAT gateway
$pip = New-AzPublicIpAddress -Name 'myPublicIP' `
  -ResourceGroupName 'myRG' -Location 'eastus' `
  -AllocationMethod Static -Sku Standard

# Create NAT gateway
$nat = New-AzNatGateway -Name 'myNATGateway' `
  -ResourceGroupName 'myRG' -Location 'eastus' `
  -PublicIpAddress $pip

# Associate to subnet
$vnet = Get-AzVirtualNetwork -ResourceGroupName 'myRG' -Name 'myVNet'
$subnet = Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name 'mySubnet'
$subnet.NatGateway = @{id = $nat.Id}
Set-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet @subnet
$vnet | Set-AzVirtualNetwork
```

**Limits:**
- Max 16 public IPs per NAT gateway
- Works with Standard resources only
- One NAT gateway per subnet
- Different subnets can use different NAT gateways

### Instance-Level Public IP

**Usage:** Attach public IP directly to VM's NIC for 1:1 mapping.

**Advantages:**
- Full control over IP assignment
- 1:1 relationship (clear visibility)
- Works with both Basic and Standard

**Limitations:**
- Higher cost per IP
- Less scalable for many VMs

### Load Balancer with Outbound Rules

**Usage:** Centralized outbound connectivity for multiple VMs behind load balancer.

**Advantages:**
- Shared public IP across multiple VMs (cost-effective)
- Built-in SNAT handling

**Limitations:**
- SNAT port exhaustion possible
- Complex configuration

### Service Endpoints

**Usage:** Enable private connectivity to Azure services without internet routing.

**Covered services:**
- Azure Storage
- Azure SQL Database
- Azure Database for MySQL/PostgreSQL
- Azure Key Vault
- Azure Cosmos DB
- Azure Event Hubs
- Azure Service Bus

**Configuration:**
```powershell
# Enable service endpoint on subnet
$vnet = Get-AzVirtualNetwork -ResourceGroupName 'myRG' -Name 'myVNet'
$subnet = Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name 'mySubnet'
$subnet.ServiceEndpoints.Add((New-AzServiceEndpointPolicy -ServiceEndpointPolicyDefinition @{
    Service = 'Microsoft.Storage'
}))
Set-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet @subnet
$vnet | Set-AzVirtualNetwork
```

---

## PowerShell & Azure CLI Commands Reference

### Virtual Network Management

```powershell
# Create VNet with subnet
New-AzVirtualNetwork -ResourceGroupName 'myRG' -Location 'eastus' `
  -Name 'myVNet' -AddressPrefix '10.0.0.0/16' `
  -Subnet (New-AzVirtualNetworkSubnetConfig -Name 'mySubnet' -AddressPrefix '10.0.0.0/24')

# Get VNet details
Get-AzVirtualNetwork -ResourceGroupName 'myRG' -Name 'myVNet'

# Update VNet (add address space)
$vnet = Get-AzVirtualNetwork -ResourceGroupName 'myRG' -Name 'myVNet'
$vnet.AddressSpace.AddressPrefixes.Add('10.1.0.0/16')
Set-AzVirtualNetwork -VirtualNetwork $vnet

# Delete VNet
Remove-AzVirtualNetwork -ResourceGroupName 'myRG' -Name 'myVNet' -Force
```

### Subnet Management

```powershell
# Add subnet to VNet
Add-AzVirtualNetworkSubnetConfig -Name 'mySubnet2' `
  -AddressPrefix '10.0.1.0/24' `
  -VirtualNetwork (Get-AzVirtualNetwork -ResourceGroupName 'myRG' -Name 'myVNet') |
  Set-AzVirtualNetwork

# Get subnet details
Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name 'mySubnet'

# Remove subnet
Remove-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name 'mySubnet' |
  Set-AzVirtualNetwork
```

### Peering Management

```powershell
# Create peering
Add-AzVirtualNetworkPeering -Name 'peer1' `
  -VirtualNetwork (Get-AzVirtualNetwork -ResourceGroupName 'myRG' -Name 'vnet1') `
  -RemoteVirtualNetworkId (Get-AzVirtualNetwork -ResourceGroupName 'myRG' -Name 'vnet2').Id

# Get peering details
Get-AzVirtualNetworkPeering -VirtualNetworkName 'vnet1' `
  -ResourceGroupName 'myRG' -Name 'peer1'

# Update peering settings
Set-AzVirtualNetworkPeering -VirtualNetworkPeeringConfig $peering

# Delete peering
Remove-AzVirtualNetworkPeering -VirtualNetworkName 'vnet1' `
  -ResourceGroupName 'myRG' -Name 'peer1' -Force
```

### Route Table Management

```powershell
# Create route table
New-AzRouteTable -ResourceGroupName 'myRG' -Location 'eastus' -Name 'myRouteTable'

# Add route
$routeTable = Get-AzRouteTable -ResourceGroupName 'myRG' -Name 'myRouteTable'
Add-AzRouteConfig -Name 'toFirewall' -RouteTable $routeTable `
  -AddressPrefix '0.0.0.0/0' -NextHopType 'VirtualAppliance' `
  -NextHopIpAddress '10.0.1.4' | Set-AzRouteTable

# Associate route table to subnet
$vnet = Get-AzVirtualNetwork -ResourceGroupName 'myRG' -Name 'myVNet'
$subnet = Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name 'mySubnet'
$subnet.RouteTable = @{id = (Get-AzRouteTable -ResourceGroupName 'myRG' -Name 'myRouteTable').Id}
Set-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet @subnet | Set-AzVirtualNetwork

# View effective routes
Get-AzEffectiveRouteTable -ResourceGroupName 'myRG' -NetworkInterfaceName 'myNIC'
```

### DNS and NAT Gateway

```powershell
# Set custom DNS servers
$vnet = Get-AzVirtualNetwork -ResourceGroupName 'myRG' -Name 'myVNet'
$vnet.DhcpOptions.DnsServers = @('10.0.1.4', '10.0.1.5')
Set-AzVirtualNetwork -VirtualNetwork $vnet

# Create NAT gateway
$pip = New-AzPublicIpAddress -Name 'pip' -ResourceGroupName 'myRG' `
  -Location 'eastus' -AllocationMethod Static -Sku Standard
$nat = New-AzNatGateway -Name 'nat' -ResourceGroupName 'myRG' `
  -Location 'eastus' -PublicIpAddress $pip

# Associate NAT gateway to subnet
$vnet = Get-AzVirtualNetwork -ResourceGroupName 'myRG' -Name 'myVNet'
$subnet = Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name 'mySubnet'
$subnet.NatGateway = @{id = $nat.Id}
Set-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet @subnet | Set-AzVirtualNetwork
```

---

## Exam Tips

### Key Concepts to Know

1. **Address Space Planning:** Non-overlapping spaces, RFC 1918, considering growth
2. **Subnet Sizing:** Reserve for scaling, account for 5 reserved IPs per subnet
3. **Peering vs. VPN:** Use peering for VNet-to-VNet (lower cost, private); use VPN for on-prem
4. **Service Endpoints:** Enable service-specific private connectivity without NAT
5. **UDRs and Routing:** Longest prefix match, next hop types, 0.0.0.0/0 implications
6. **DNS Strategy:** Private zones preferred for managed domains; custom servers for hybrid
7. **Outbound Connectivity:** NAT Gateway recommended (static IPs, no SNAT exhaustion)
8. **Hub-and-Spoke:** Centralize services, use forwarded traffic + UDRs for transit
9. **Global Peering:** Works with Standard resources; Basic Load Balancer unsupported
10. **Transit Peering:** Enable "AllowForwardedTraffic" and route via NVA for inspection

### Common Scenarios

| Scenario | Solution |
|----------|----------|
| **Multi-region deployment** | Global VNet peering + Private DNS zones |
| **On-prem connectivity** | VPN Gateway + UDRs for routing |
| **Centralized security** | Hub-and-spoke with NVA + UDRs |
| **Service access** | Service endpoints + private connectivity |
| **Outbound static IP** | NAT Gateway on subnet |
| **Internal DNS** | Azure Private DNS Zones with autoregistration |
| **Traffic inspection** | Hub NVA with spoke UDRs pointing to NVA |

---

## Exam Scenarios

### Scenario 1: Multi-Region Hub-and-Spoke Network

**Requirements:**
- Headquarters in East US; branch office in West US
- Centralized firewall in East US hub
- Spokes connect to hub; branch VNet also connects
- All traffic through hub for inspection
- Static public IP for outbound traffic

**Configuration:**
1. Create hub VNet (10.0.0.0/16) in East US with firewall subnet
2. Create spoke VNets (10.1.0.0/16, 10.2.0.0/16) in East US
3. Create branch VNet (192.168.0.0/16) in West US
4. Peer hub-to-spoke (local), hub-to-branch (global)
5. Enable "AllowForwardedTraffic" on all peerings
6. Create route tables in spokes with 0.0.0.0/0 → firewall NVA IP
7. Deploy firewall in hub's firewall subnet
8. Attach NAT Gateway with static public IP to hub's internet-facing subnet
9. Configure firewall outbound rules; enable IP forwarding on NVA NIC
10. Set branch's default route (0.0.0.0/0) to hub's NVA via VPN or global peering

**Exam Validation Points:**
- Peering configuration (bidirectional, traffic allowed)
- Route precedence (longer prefix matches)
- Hub as single point of inspection
- NAT gateway static outbound IP
- Global peering latency acceptable for branch traffic

### Scenario 2: Private Database Access with Service Endpoint

**Requirements:**
- Web app and database both private (no public IPs)
- Database access only from app subnet
- Compliance: no internet routing to database
- Multiple app subnets; single database subnet

**Configuration:**
1. Create VNet (10.0.0.0/16) with subnets:
   - App subnet 1 (10.0.1.0/24)
   - App subnet 2 (10.0.2.0/24)
   - Database subnet (10.0.3.0/24)
2. Create Azure SQL Database in database subnet
3. Enable Microsoft.Sql service endpoint on database subnet
4. Create Private DNS Zone for `database.windows.net`
5. Link VNet to Private DNS Zone with autoregistration
6. Update app connection strings to use Private DNS name
7. Configure SQL firewall to:
   - Allow service endpoints
   - Deny public network access
   - Add VNet rules for app subnets
8. Deploy web app instances in app subnets
9. Configure NSGs to allow traffic only between app and database subnets

**Exam Validation Points:**
- Service endpoint vs. NAT gateway difference
- Private DNS zone configuration
- SQL firewall rules with VNet endpoints
- NSG ingress/egress rules enforcing traffic policy

### Scenario 3: Hybrid Connectivity with On-Premises DNS

**Requirements:**
- Azure VNet (10.0.0.0/16) with domain-joined VMs
- On-premises network (192.168.0.0/16) with Active Directory DC
- Name resolution both ways (Azure → on-prem AD, on-prem → Azure resources)
- Redundancy: multiple DNS forwarders

**Configuration:**
1. Create Azure VNet (10.0.0.0/16)
2. Create VPN connection to on-premises (site-to-site VPN)
3. Deploy domain-joined DNS forwarder VMs in Azure (10.0.1.0/24):
   - Two VMs for redundancy
   - Conditional forwarding rules → on-prem DC (192.168.1.4)
   - Forwarding rules for on-prem domain names
   - Forward external queries to 8.8.8.8 (public DNS)
4. Set Azure VNet custom DNS servers to forwarder IPs
5. On-premises: configure DC to forward Azure-bound queries to Azure forwarder
6. Verify bi-directional name resolution:
   - From Azure VM: `nslookup server.corpnet.local` (on-prem resource)
   - From on-prem DC: `nslookup vmname.internal.cloudapp.net` (Azure VM)
7. Create Private DNS Zone for Azure internal domain names (optional, for cleaner naming)

**Exam Validation Points:**
- Conditional forwarding rules for hybrid DNS
- Firewall allowing DNS ports (53) between networks
- Forwarding timeout > 4 seconds
- Redundancy via multiple forwarders

---

## Troubleshooting Virtual Networking

### VMs Cannot Ping Each Other

**Causes:**
1. VMs in different subnets, no routes between them (expected, use communication)
2. Network Security Group (NSG) blocking ICMP traffic
3. VMs in peered VNets, peering not created or not active
4. UDR with next hop "None" blocking traffic
5. NVA not forwarding traffic (IP forwarding disabled)

**Troubleshooting Steps:**
```powershell
# 1. Check VNet and subnet configuration
Get-AzVirtualNetwork -ResourceGroupName 'myRG' -Name 'myVNet'
Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet

# 2. View effective routes on VM NIC
Get-AzEffectiveRouteTable -ResourceGroupName 'myRG' -NetworkInterfaceName 'vm1-nic'

# 3. Check NSG rules
Get-AzNetworkSecurityGroup -ResourceGroupName 'myRG' -Name 'myNSG' | 
  Get-AzNetworkSecurityRuleConfig

# 4. Verify peering status (if VNets peered)
Get-AzVirtualNetworkPeering -VirtualNetworkName 'vnet1' -ResourceGroupName 'myRG'

# 5. Check IP forwarding on NVA (if routing through appliance)
Get-AzNetworkInterface -ResourceGroupName 'myRG' -Name 'nva-nic' | 
  Select-Object -ExpandProperty EnableIPForwarding
```

### Connectivity to Azure Services Not Working

**Causes:**
1. Service endpoint not enabled on subnet
2. Firewall rules blocking traffic
3. Private endpoint not created for service
4. DNS resolution failing (FQDN not resolving)

**Solutions:**
- Enable service endpoint on subnet for the specific service
- Create private endpoint for private connectivity
- Verify DNS settings (Private DNS Zone linked)
- Check storage account firewall rules
- Ensure Standard SKU for services requiring it

### Peering Not Working

**For same-region peering:**
- Verify peering created bidirectionally
- Check peering status = "Connected"
- Ensure address spaces don't overlap
- Verify "AllowVirtualNetworkAccess" enabled
- Check NSG rules on both sides

**For cross-region (global) peering:**
- Confirm both VNets in different regions
- Verify peering status = "Connected"
- Check if using Basic Load Balancer (unsupported)
- Verify forwarded traffic setting if using NVA
- Check bandwidth (cross-region higher latency expected)

### DNS Name Resolution Issues

**Cannot resolve VM by hostname:**
```powershell
# Verify DNS servers configured on VNet
$vnet = Get-AzVirtualNetwork -ResourceGroupName 'myRG' -Name 'myVNet'
$vnet.DhcpOptions.DnsServers

# Test DNS from VM
nslookup vm1.internal.cloudapp.net
nslookup vm1.contoso.com # if using Private DNS Zone

# Check custom DNS server accessibility
Test-NetConnection -ComputerName 10.0.1.4 -Port 53
```

**Fix:** Verify DNS servers (Azure default or custom) configured correctly; ensure custom servers accessible and forwarding enabled.

### NAT Gateway Not Working

**Causes:**
1. NAT gateway not associated with subnet
2. No public IP or prefix assigned to NAT gateway
3. Outbound traffic redirected elsewhere (UDR, firewall rule)
4. NSG blocking outbound traffic

**Verification:**
```powershell
# Verify NAT gateway association
$vnet = Get-AzVirtualNetwork -ResourceGroupName 'myRG' -Name 'myVNet'
$subnet = Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name 'mySubnet'
$subnet.NatGateway

# Verify NAT gateway has public IP
$nat = Get-AzNatGateway -ResourceGroupName 'myRG' -Name 'myNAT'
$nat.PublicIpAddresses
```

---

## Performance and Best Practices

### Performance Optimization

| Aspect | Best Practice |
|--------|---------------|
| **Latency** | Use peering within region; global peering for cross-region (acceptable latency) |
| **Throughput** | Ensure VMs not behind Basic Load Balancer (when using peering) |
| **DNS** | Use Private DNS Zones for sub-100ms name resolution |
| **Routing** | Minimize UDR complexity; use Azure-provided routes when possible |
| **NAT Gateway** | Multiple public IPs/prefixes for high throughput outbound traffic |

### Security Best Practices

1. **Always use NSGs** for ingress/egress filtering
2. **Use service endpoints** instead of public connectivity when possible
3. **Prefer Private DNS Zones** over custom DNS servers
4. **Enable VNet flow logs** for traffic monitoring
5. **Use hub-and-spoke** for centralized security enforcement
6. **IP Forwarding:** Only enable on NVAs that need it
7. **Public IP:** Minimize public IPs; use Private Link for service access
8. **Segmentation:** Use multiple subnets; don't use single large subnet

### Scaling Considerations

- **Address space:** Plan for future growth; CIDR expansion possible but requires careful planning
- **VNet limits:** Monitor peering count (500 default, 1000 with Virtual Network Manager)
- **Route tables:** Keep routes manageable; use service tags for bulk address handling
- **DNS:** Private DNS Zones scale automatically; custom servers require HA planning
- **NAT Gateway:** Add multiple public IPs as throughput increases

