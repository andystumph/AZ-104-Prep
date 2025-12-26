# Integrate On-Premises Networks with Azure

> **Exam Weight:** Networking 15-20% (hybrid connectivity is critical for real-world Azure deployments)

## Table of Contents
- [Overview](#overview)
- [VPN Gateway (Site-to-Site)](#vpn-gateway-site-to-site)
  - [VPN Gateway SKUs](#vpn-gateway-skus)
  - [Active-Active Configuration](#active-active-configuration)
  - [BGP with VPN Gateway](#bgp-with-vpn-gateway)
  - [Create Site-to-Site VPN](#create-site-to-site-vpn)
  - [PowerShell and CLI (VPN Gateway)](#powershell-and-cli-vpn-gateway)
- [Point-to-Site VPN](#point-to-site-vpn)
  - [Authentication Methods](#authentication-methods)
  - [Configure P2S VPN](#configure-p2s-vpn)
- [ExpressRoute](#expressroute)
  - [ExpressRoute SKUs and Bandwidth](#expressroute-skus-and-bandwidth)
  - [Private Peering vs Microsoft Peering](#private-peering-vs-microsoft-peering)
  - [ExpressRoute FastPath](#expressroute-fastpath)
  - [ExpressRoute Global Reach](#expressroute-global-reach)
  - [High Availability and Resilience](#high-availability-and-resilience)
  - [Create ExpressRoute Connection](#create-expressroute-connection)
  - [PowerShell and CLI (ExpressRoute)](#powershell-and-cli-expressroute)
- [Azure Virtual WAN](#azure-virtual-wan)
  - [Virtual WAN Architecture](#virtual-wan-architecture)
  - [Routing Intent](#routing-intent)
  - [Configure Virtual WAN](#configure-virtual-wan)
- [Comparison Matrix](#comparison-matrix)
- [Best Practices](#best-practices)
- [Exam Tips](#exam-tips)
- [Scenarios](#scenarios)
- [Troubleshooting](#troubleshooting)
- [Command Quick Reference](#command-quick-reference)

---

## Overview

Hybrid connectivity connects on-premises networks to Azure using **VPN Gateway** (encrypted over internet), **ExpressRoute** (private dedicated circuits), or **Azure Virtual WAN** (global transit hub). Selection depends on bandwidth, latency, security, and cost requirements.

---

## VPN Gateway (Site-to-Site)

### VPN Gateway SKUs

| SKU | Bandwidth | Tunnels | BGP | Availability Zones | Use Case |
|-----|-----------|---------|-----|-------------------|----------|
| **Basic** | 100 Mbps | 10 (S2S), no P2S | No | No | Dev/test only |
| **VpnGw1** | 650 Mbps | 30 | Yes | No | Small production |
| **VpnGw2** | 1 Gbps | 30 | Yes | No | Medium production |
| **VpnGw3** | 1.25 Gbps | 30 | Yes | No | Large production |
| **VpnGw1AZ** | 650 Mbps | 30 | Yes | Yes | Zone-redundant small |
| **VpnGw2AZ** | 1 Gbps | 30 | Yes | Yes | Zone-redundant medium |
| **VpnGw3AZ** | 1.25 Gbps | 30 | Yes | Yes | Zone-redundant large |
| **VpnGw4AZ** | 2.3 Gbps | 100 | Yes | Yes | Very large production |
| **VpnGw5AZ** | 2.3 Gbps (HA tunnels) | 100 | Yes | Yes | Maximum throughput |

> Use AZ SKUs for production; Basic SKU lacks BGP, zones, and production SLA.

### Active-Active Configuration

- Two gateway instances with two public IPs.
- Provides automatic failover; both tunnels active simultaneously.
- Requires route-based VPN; on-prem device must support two tunnels.

### BGP with VPN Gateway

- **Benefits:** Dynamic routing, automatic failover, multi-site transit routing.
- **ASN:** Must specify ASN for VPN Gateway (default 65515 if not specified); avoid reserved ASNs (8074, 8075, 12076, 65515-65520).
- **BGP Peer IP:** Automatically allocated from GatewaySubnet range.
- **Route Advertisement:** Gateway advertises VNet prefixes; on-prem advertises local prefixes.

### Create Site-to-Site VPN

**Prerequisites:**
- VNet with GatewaySubnet (/27 or larger recommended).
- Local Network Gateway (on-prem IP + address prefixes).
- Pre-shared key (PSK) for IPsec.

**Portal Summary:**
1. Create **Virtual Network Gateway** → VPN type, route-based, SKU (VpnGw1AZ+), enable active-active if needed, configure BGP ASN if using BGP.
2. Create **Local Network Gateway** → on-prem public IP, address space, BGP peer IP (if BGP).
3. Create **Connection** → Site-to-Site (IPsec), link VNG to LNG, enter shared key, enable BGP if configured.
4. Configure **on-prem VPN device** with Azure public IP, PSK, IKEv2/IPsec policies.

### PowerShell and CLI (VPN Gateway)

```powershell
# Variables
$rg="myRG"; $loc="eastus"; $vnetName="hubVNet"; $gwSubnetPrefix="10.0.255.0/27"

# GatewaySubnet
$vnet = Get-AzVirtualNetwork -Name $vnetName -ResourceGroupName $rg
Add-AzVirtualNetworkSubnetConfig -Name "GatewaySubnet" -VirtualNetwork $vnet -AddressPrefix $gwSubnetPrefix
$vnet | Set-AzVirtualNetwork

# Public IP for VPN Gateway
$gwpip = New-AzPublicIpAddress -Name "vpnGwPip" -ResourceGroupName $rg -Location $loc -AllocationMethod Static -Sku Standard -Zone 1,2,3

# Gateway IP config
$subnet = Get-AzVirtualNetworkSubnetConfig -Name "GatewaySubnet" -VirtualNetwork $vnet
$gwipconfig = New-AzVirtualNetworkGatewayIpConfig -Name "gwipconfig" -SubnetId $subnet.Id -PublicIpAddressId $gwpip.Id

# Create VPN Gateway with BGP
New-AzVirtualNetworkGateway -Name "vpnGw" -ResourceGroupName $rg -Location $loc -IpConfigurations $gwipconfig -GatewayType Vpn -VpnType RouteBased -GatewaySku VpnGw1AZ -Asn 65010 -EnableBgp $true

# Local Network Gateway (on-prem)
New-AzLocalNetworkGateway -Name "onPremLng" -ResourceGroupName $rg -Location $loc -GatewayIpAddress "203.0.113.1" -AddressPrefix "192.168.0.0/16" -Asn 65020 -BgpPeeringAddress "192.168.255.254"

# Create Connection
$vng = Get-AzVirtualNetworkGateway -Name "vpnGw" -ResourceGroupName $rg
$lng = Get-AzLocalNetworkGateway -Name "onPremLng" -ResourceGroupName $rg
New-AzVirtualNetworkGatewayConnection -Name "hub-to-onprem" -ResourceGroupName $rg -Location $loc -VirtualNetworkGateway1 $vng -LocalNetworkGateway2 $lng -ConnectionType IPsec -SharedKey "AzureSharedKey123!" -EnableBgp $true
```

```bash
# Azure CLI: Create VPN Gateway with BGP
rg=myRG
loc=eastus
vnet=hubVNet
gwSubnet=10.0.255.0/27
onPremIP=203.0.113.1

# Create GatewaySubnet
az network vnet subnet create --resource-group $rg --vnet-name $vnet --name GatewaySubnet --address-prefix $gwSubnet

# Public IP
az network public-ip create --resource-group $rg --name vpnGwPip --sku Standard --allocation-method Static --location $loc --zone 1 2 3

# VPN Gateway
az network vnet-gateway create \
  --resource-group $rg \
  --name vpnGw \
  --vnet $vnet \
  --public-ip-address vpnGwPip \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw1AZ \
  --asn 65010 \
  --bgp-peering-address 10.0.255.254

# Local Network Gateway
az network local-gateway create \
  --resource-group $rg \
  --name onPremLng \
  --gateway-ip-address $onPremIP \
  --local-address-prefixes 192.168.0.0/16 \
  --asn 65020 \
  --bgp-peering-address 192.168.255.254

# Connection
az network vpn-connection create \
  --resource-group $rg \
  --name hub-to-onprem \
  --vnet-gateway1 vpnGw \
  --local-gateway2 onPremLng \
  --shared-key "AzureSharedKey123!" \
  --enable-bgp
```

---

## Point-to-Site VPN

### Authentication Methods

| Method | Description | Use Case |
|--------|-------------|----------|
| **Azure Certificate** | Self-signed or CA-issued root cert uploaded to gateway; client certs on devices | Simple, no external dependencies |
| **RADIUS - Certificate** | RADIUS server validates client cert | Enterprise PKI integration |
| **RADIUS - Password/MFA** | RADIUS with AD/NPS for username/password + MFA | Enterprise auth, Active Directory |
| **Microsoft Entra ID** | Azure AD authentication with conditional access | Cloud-native, SSO, MFA |

### Configure P2S VPN

**Prerequisites:**
- VPN Gateway (route-based, supports P2S).
- Client address pool (non-overlapping with VNet or on-prem).
- Root certificate (for Azure cert auth) or RADIUS server (for RADIUS auth).

**Portal Summary:**
1. Navigate to **Virtual Network Gateway** → **Point-to-site configuration**.
2. Set **Address pool** (e.g., 172.16.0.0/24).
3. Choose **Tunnel type** (IKEv2, OpenVPN, or both).
4. Select **Authentication type** (Azure certificate, RADIUS, Azure AD).
5. Upload **Root certificate** public key (Base64 .cer) if using cert auth.
6. For RADIUS: Enter RADIUS server IP and shared secret.
7. Save configuration.
8. Download **VPN client** package; install on client devices.

---

## ExpressRoute

### ExpressRoute SKUs and Bandwidth

| SKU | Geography | Bandwidth Options | Use Case |
|-----|-----------|-------------------|----------|
| **Local** | Metro area only | 50 Mbps - 10 Gbps | Low latency within metro |
| **Standard** | Within geopolitical region | 50 Mbps - 10 Gbps | Regional connectivity |
| **Premium** | Global (any Azure region) | 50 Mbps - 10 Gbps | Global reach, Global Reach feature |

### Private Peering vs Microsoft Peering

| Aspect | Private Peering | Microsoft Peering |
|--------|----------------|-------------------|
| **Purpose** | Access Azure IaaS (VMs, VNets) | Access Azure PaaS (Storage, SQL, M365) |
| **IP Addressing** | Private IPs (RFC 1918) | Public IPs (owned or provider) |
| **BGP Required** | Yes | Yes |
| **Subnet Size** | /30 (primary + secondary) | /30 (primary + secondary) |
| **VLAN ID** | Customer-specified | Customer-specified |
| **Route Filter** | No | Yes (for specific services) |

### ExpressRoute FastPath

- Bypasses virtual network gateway for data plane traffic.
- Reduces latency by sending traffic directly to VMs.
- Requires **Ultra Performance** or **ErGw3AZ** gateway SKU.
- Gateway still required for route programming.

### ExpressRoute Global Reach

- Connects two ExpressRoute circuits to enable on-prem-to-on-prem via Azure backbone.
- Requires Premium SKU.
- Uses /29 subnet for peering between circuits.

### High Availability and Resilience

**Recommended Patterns:**
1. **Standard Resiliency:** Single circuit with two connections (active-active) at one peering location.
2. **High Resiliency:** Two circuits at different peering locations in same metro.
3. **Maximum Resiliency:** Multiple circuits across different metros/regions + VPN backup.

**Resilience Best Practices:**
- Use zone-redundant gateway SKUs (ErGw1AZ, ErGw2AZ, ErGw3AZ).
- Configure site-to-site VPN as backup over Microsoft peering.
- Different service providers for each circuit.
- Monitor circuit health (BGP availability, bits in/out).

### Create ExpressRoute Connection

**Prerequisites:**
- ExpressRoute circuit provisioned by provider (status: Provisioned).
- Virtual network with GatewaySubnet.
- ExpressRoute virtual network gateway.

**Portal Summary:**
1. Create **Virtual Network Gateway** → Gateway type: ExpressRoute, SKU (ErGw1AZ+).
2. Navigate to **ExpressRoute circuit** → **Peerings** → **Azure private peering**.
3. Configure **Peer ASN**, **Primary subnet** (/30), **Secondary subnet** (/30), **VLAN ID**.
4. Create **Connection** between VNet Gateway and ExpressRoute circuit.
5. Verify BGP session established (check peering status).

### PowerShell and CLI (ExpressRoute)

```powershell
# Variables
$rg="myRG"; $loc="eastus"; $vnetName="prodVNet"

# Create ExpressRoute Gateway
$gwpip = New-AzPublicIpAddress -Name "erGwPip" -ResourceGroupName $rg -Location $loc -AllocationMethod Static -Sku Standard -Zone 1,2,3
$vnet = Get-AzVirtualNetwork -Name $vnetName -ResourceGroupName $rg
$subnet = Get-AzVirtualNetworkSubnetConfig -Name "GatewaySubnet" -VirtualNetwork $vnet
$gwipconfig = New-AzVirtualNetworkGatewayIpConfig -Name "ergwipconfig" -SubnetId $subnet.Id -PublicIpAddressId $gwpip.Id

New-AzVirtualNetworkGateway -Name "erGw" -ResourceGroupName $rg -Location $loc -IpConfigurations $gwipconfig -GatewayType ExpressRoute -GatewaySku ErGw1AZ

# Configure Private Peering (assumes circuit exists)
$circuit = Get-AzExpressRouteCircuit -Name "myCircuit" -ResourceGroupName $rg
Add-AzExpressRouteCircuitPeeringConfig -Name "AzurePrivatePeering" -ExpressRouteCircuit $circuit -PeeringType AzurePrivatePeering -PeerASN 65020 -PrimaryPeerAddressPrefix "10.0.0.0/30" -SecondaryPeerAddressPrefix "10.0.0.4/30" -VlanId 200
Set-AzExpressRouteCircuit -ExpressRouteCircuit $circuit

# Create Connection
$gw = Get-AzVirtualNetworkGateway -Name "erGw" -ResourceGroupName $rg
$circuit = Get-AzExpressRouteCircuit -Name "myCircuit" -ResourceGroupName $rg
New-AzVirtualNetworkGatewayConnection -Name "er-connection" -ResourceGroupName $rg -Location $loc -VirtualNetworkGateway1 $gw -PeerId $circuit.Id -ConnectionType ExpressRoute
```

```bash
# Azure CLI: Create ExpressRoute Gateway
rg=myRG
loc=eastus
vnet=prodVNet
circuit=myCircuit

# Public IP
az network public-ip create --resource-group $rg --name erGwPip --sku Standard --allocation-method Static --location $loc --zone 1 2 3

# ExpressRoute Gateway
az network vnet-gateway create \
  --resource-group $rg \
  --name erGw \
  --vnet $vnet \
  --public-ip-address erGwPip \
  --gateway-type ExpressRoute \
  --sku ErGw1AZ

# Configure Private Peering
az network express-route peering create \
  --circuit-name $circuit \
  --resource-group $rg \
  --peering-type AzurePrivatePeering \
  --peer-asn 65020 \
  --primary-peer-subnet 10.0.0.0/30 \
  --secondary-peer-subnet 10.0.0.4/30 \
  --vlan-id 200

# Create Connection
az network vpn-connection create \
  --resource-group $rg \
  --name er-connection \
  --vnet-gateway1 erGw \
  --express-route-circuit2 $circuit \
  --location $loc
```

---

## Azure Virtual WAN

### Virtual WAN Architecture

- **Hub:** Managed by Microsoft; contains VPN/ExpressRoute/P2S gateways, routing, firewall.
- **Spokes:** VNets connected to hub; transitive connectivity enabled.
- **Branches:** On-prem sites connected via VPN/ExpressRoute.
- **Routing Intent:** Simplifies routing policies (Internet, Private traffic).

### Routing Intent

**Intent Types:**
1. **Internet Traffic:** Routes internet-bound traffic to Azure Firewall/NVA.
2. **Private Traffic:** Routes VNet/branch traffic through security appliance.

**Benefits:**
- Centralized security enforcement.
- Simplified routing (no UDRs in spokes).
- Automatic route propagation.

### Configure Virtual WAN

**Portal Summary:**
1. Create **Virtual WAN** (Standard SKU for S2S/ER/P2S).
2. Create **Virtual Hub** in region; specify address space (e.g., 10.1.0.0/16).
3. Add **VPN Gateway** to hub (scale units for bandwidth).
4. Create **VPN site** for each branch (public IP, BGP settings, link info).
5. Create **VPN connection** between site and hub.
6. Connect **VNets** to hub.
7. (Optional) Configure **Routing Intent** for Azure Firewall/NVA.

---

## Comparison Matrix

| Feature | VPN Gateway | ExpressRoute | Virtual WAN |
|---------|-------------|--------------|-------------|
| **Connectivity** | Encrypted over internet | Private dedicated circuit | Hub-spoke with VPN/ER |
| **Bandwidth** | Up to 2.3 Gbps (VpnGw5AZ) | Up to 10 Gbps | Aggregate (scale units) |
| **Latency** | Variable (internet) | Consistent, low | Low (backbone routing) |
| **Cost** | Gateway + egress | Circuit + egress + gateway | Hub + connections + egress |
| **Setup Time** | Minutes to hours | Weeks (provider provisioning) | Minutes (hub); weeks (ER) |
| **BGP** | Optional | Required | Supported |
| **Resilience** | Active-active, multiple tunnels | Dual circuits, Global Reach | Multi-hub, automatic failover |
| **Use Case** | Small-medium sites, backup | Mission-critical, high throughput | Global enterprise, multi-region |

---

## Best Practices

- **VPN Gateway:** Use VpnGw1AZ+ SKUs; enable BGP; configure active-active for HA.
- **ExpressRoute:** Use Premium for global reach; deploy zone-redundant gateways; configure VPN backup.
- **Monitoring:** Enable diagnostic logs, Connection Monitor, NSG flow logs; alert on BGP/tunnel status.
- **Security:** Use forced tunneling for VPN; apply NSGs; implement Private Link for PaaS services.
- **Scalability:** Virtual WAN for multi-site; route intent for centralized policies; scale units for bandwidth.

---

## Exam Tips

- **VPN Gateway SKUs:** Basic = no BGP/zones; AZ SKUs = zone-redundant; VpnGw5AZ = HA tunnels.
- **BGP:** Required for ExpressRoute; optional for VPN; enables dynamic routing and transit.
- **ExpressRoute Peering:** Private (VNets), Microsoft (PaaS/M365).
- **FastPath:** Bypasses gateway for data plane; requires Ultra Performance/ErGw3AZ.
- **Global Reach:** Links ER circuits for on-prem-to-on-prem; Premium SKU only.
- **Virtual WAN:** Managed hub; transitive connectivity; routing intent simplifies policies.
- **Resilience:** Standard (single site) < High (metro diversity) < Maximum (multi-region + VPN).

---

## Scenarios

### Scenario 1: SMB with Hybrid Connectivity

- **Requirement:** 100 Mbps, encrypted, cost-effective.
- **Solution:** Site-to-Site VPN with VpnGw1 SKU; BGP for automatic failover; single tunnel sufficient.

### Scenario 2: Enterprise with Mission-Critical Apps

- **Requirement:** 1 Gbps, low latency, SLA 99.95%.
- **Solution:** ExpressRoute Standard (1 Gbps circuit); private peering; ErGw2AZ gateway; VPN backup over Microsoft peering.

### Scenario 3: Global Corp with Multi-Region Presence

- **Requirement:** Connect 50+ branches, 10+ Azure regions, centralized firewall.
- **Solution:** Azure Virtual WAN (Standard); hubs in each region; VPN connections for branches; ExpressRoute for primary datacenters; routing intent with Azure Firewall.

### Scenario 4: Remote Workers Accessing Azure

- **Requirement:** 500 remote users; Azure AD SSO; MFA.
- **Solution:** Point-to-Site VPN with Microsoft Entra ID authentication; OpenVPN protocol; VpnGw2AZ gateway (supports 500+ connections).

### Scenario 5: Disaster Recovery with Hybrid

- **Requirement:** ExpressRoute primary; VPN failover.
- **Solution:** ExpressRoute circuit + private peering; Site-to-Site VPN over Microsoft peering as backup; BGP to prefer ER; automatic failover if ER down.

---

## Troubleshooting

### VPN Tunnel Down

- **Causes:** PSK mismatch, IKE/IPsec policy mismatch, on-prem device offline, NSG blocking UDP 500/4500.
- **Actions:** Verify PSK matches on both sides; check `TunnelDiagnosticLog`; test IKE phase 1/2; ensure NSG allows VPN traffic; validate on-prem public IP.

### ExpressRoute BGP Session Down

- **Causes:** /30 subnet misconfiguration, VLAN ID mismatch, peer ASN incorrect, BGP auth key mismatch.
- **Actions:** Verify `/30` subnets (Azure uses second IP); confirm VLAN ID on circuit and on-prem; check `BGPRouteLog` for route advertisements; validate MD5 key if configured.

### Virtual WAN Connection Fails

- **Causes:** Hub address space overlap with spoke, VPN site config error, routing intent misconfiguration.
- **Actions:** Check address space uniqueness; verify VPN site public IP reachable; review routing policies in hub; confirm connection status in portal.

### High Latency on VPN

- **Causes:** Internet congestion, MTU/MSS mismatch, encryption overhead.
- **Actions:** Test latency to Azure public IP; adjust MTU/MSS (recommended 1350-1400); consider ExpressRoute for consistent latency.

---

## Command Quick Reference

```powershell
# VPN Gateway: Get connection status
Get-AzVirtualNetworkGatewayConnection -Name "hub-to-onprem" -ResourceGroupName "myRG" | Select-Object Name, ConnectionStatus, EgressBytesTransferred, IngressBytesTransferred

# ExpressRoute: Verify peering config
$circuit = Get-AzExpressRouteCircuit -Name "myCircuit" -ResourceGroupName "myRG"
Get-AzExpressRouteCircuitPeeringConfig -Name "AzurePrivatePeering" -ExpressRouteCircuit $circuit

# Enable VPN Gateway BGP
$gw = Get-AzVirtualNetworkGateway -Name "vpnGw" -ResourceGroupName "myRG"
Set-AzVirtualNetworkGateway -VirtualNetworkGateway $gw -Asn 65010 -EnableBgp $true

# ExpressRoute Global Reach (link two circuits)
$circuit1 = Get-AzExpressRouteCircuit -Name "circuit1" -ResourceGroupName "myRG"
$circuit2 = Get-AzExpressRouteCircuit -Name "circuit2" -ResourceGroupName "myRG"
Add-AzExpressRouteCircuitConnectionConfig -Name "globalreach" -ExpressRouteCircuit $circuit1 -PeerExpressRouteCircuitPeering $circuit2.Peerings[0].Id -AddressPrefix "10.0.0.0/29"
Set-AzExpressRouteCircuit -ExpressRouteCircuit $circuit1
```

```bash
# CLI: VPN connection status
az network vpn-connection show --resource-group myRG --name hub-to-onprem --query "{status:connectionStatus, ingress:ingressBytesTransferred, egress:egressBytesTransferred}"

# ExpressRoute: Show peering details
az network express-route peering show --circuit-name myCircuit --resource-group myRG --name AzurePrivatePeering

# Virtual WAN: Create VPN site
az network vpn-site create \
  --resource-group myRG \
  --name branch1 \
  --location eastus \
  --virtual-wan myVWAN \
  --ip-address 203.0.113.1 \
  --asn 65030 \
  --bgp-peering-address 192.168.1.1 \
  --address-prefixes 192.168.0.0/16

# Virtual WAN: Connect VPN site to hub
az network vpn-gateway connection create \
  --resource-group myRG \
  --gateway-name hubVpnGw \
  --name branch1-connection \
  --remote-vpn-site branch1 \
  --shared-key "AzureKey123!" \
  --enable-bgp true
```

---

**Quick Recap**: Use **VPN Gateway** for encrypted internet-based connectivity with BGP for dynamic routing; use **ExpressRoute** for private, high-throughput connections with private/Microsoft peering; use **Virtual WAN** for global hub-spoke with centralized routing and security. Configure resilience with active-active, multiple circuits, and VPN backup.
