# Configure Load Balancing

> **Exam Weight:** Networking 15-20% (load balancing and traffic management are frequent exam scenarios)

## Table of Contents
- [Overview](#overview)
- [Azure Load Balancer (Layer 4)](#azure-load-balancer-layer-4)
	- [SKU Comparison: Basic vs Standard](#sku-comparison-basic-vs-standard)
	- [Key Components](#key-components)
	- [Load Balancing Rules](#load-balancing-rules)
	- [NAT Rules and Outbound Rules](#nat-rules-and-outbound-rules)
	- [Health Probes](#health-probes)
	- [High Availability Ports](#high-availability-ports)
	- [Limitations](#limitations)
	- [Create a Public Load Balancer](#create-a-public-load-balancer)
	- [Create an Internal Load Balancer](#create-an-internal-load-balancer)
	- [PowerShell and CLI Examples (Load Balancer)](#powershell-and-cli-examples-load-balancer)
- [Application Gateway (Layer 7)](#application-gateway-layer-7)
	- [When to Use Application Gateway](#when-to-use-application-gateway)
	- [SKU Overview](#sku-overview)
	- [Key Concepts](#key-concepts)
	- [Web Application Firewall (WAF)](#web-application-firewall-waf)
	- [Create an Application Gateway](#create-an-application-gateway)
	- [PowerShell and CLI Examples (App Gateway)](#powershell-and-cli-examples-app-gateway)
- [Traffic Manager vs Front Door vs Gateway vs Load Balancer](#traffic-manager-vs-front-door-vs-gateway-vs-load-balancer)
- [Design Decision Matrix](#design-decision-matrix)
- [Best Practices](#best-practices)
- [Exam Tips](#exam-tips)
- [Scenarios](#scenarios)
- [Troubleshooting](#troubleshooting)
- [Command Quick Reference](#command-quick-reference)

---

## Overview

Azure provides multiple load balancing options across Layers 4 and 7. **Azure Load Balancer** is a high-performance, ultra-low latency Layer 4 balancer for TCP/UDP. **Application Gateway** is a Layer 7 reverse proxy with WAF, SSL offload, and path/host-based routing. **Traffic Manager** and **Front Door** add global routing, CDN, and edge capabilities. Selecting the right service depends on protocol, routing needs, TLS handling, security, and global reach.

---

## Azure Load Balancer (Layer 4)

### SKU Comparison: Basic vs Standard

| Feature | Basic | Standard |
|---------|-------|----------|
| **Backend Instances** | Up to 300 | Up to 1000 |
| **Availability Zones** | No | Yes (zonal or zone-redundant) |
| **Scale** | Limited | Production-scale, SLA-backed |
| **Security** | Open by default | Closed by default (NSG must allow) |
| **IPv6** | No | Yes |
| **HA Ports** | No | Yes |
| **Outbound Rules** | Limited | Full outbound rules |
| **Recommended** | Dev/test only | Production |

### Key Components

- **Frontend IP configuration**: Public or private IP exposed by the Load Balancer.
- **Backend pool**: NICs/VM scale set instances receiving traffic.
- **Health probes**: TCP/HTTP/HTTPS probes determining backend availability.
- **Load balancing rules**: Map frontend to backend pool and health probe for a specific port/protocol.
- **Inbound NAT rules**: Map frontend port to specific backend instance/port (per-VM access such as RDP/SSH).
- **Outbound rules**: Define SNAT for outbound internet access from backend pool.

### Load Balancing Rules

Load Balancer distributes flows using a 5-tuple hash (source IP, source port, destination IP, destination port, protocol). Session persistence options:
- **None (hash-based)**: Default; new hash each flow.
- **Client IP**: Same source IP goes to same backend.
- **Client IP and protocol**: Includes protocol in persistence hash.

### NAT Rules and Outbound Rules

- **Inbound NAT Rules**: One-to-one mapping from frontend port to specific backend instance/port (e.g., 50001 → VM1:3389).
- **Outbound Rules**: Define SNAT for outbound internet; Standard SKU often paired with NAT Gateway for large SNAT port allocation.

### Health Probes

- Types: TCP, HTTP, HTTPS
- Probe frequency and unhealthy threshold define failure detection.
- Backend marked **Unhealthy** after consecutive failures; traffic stops flowing until probe succeeds.
- HTTP/HTTPS probes expect 200-399; other codes/timeouts mark failure.

### High Availability Ports

- Special load balancing rule that forwards **all TCP/UDP ports** to backend pool.
- Ideal for NVAs and scenarios needing full port range.
- Requires Standard SKU; one HA Ports rule per frontend.

### Limitations

- No TLS termination, no header/path-based routing, no caching, no WAF.
- Regional scope (but zone-redundant within region).

### Create a Public Load Balancer

Portal summary (Standard SKU):
1. Create **Load Balancer** → **Public** → **Standard**.
2. Add **Frontend IP** (new public IP, Standard, static recommended).
3. Add **Backend pool** (NICs or VMSS instances).
4. Configure **Health probe** (TCP 80/443 or HTTP path `/healthz`).
5. Add **Load balancing rule** (frontend port 80 → backend port 80, probe, session persistence if needed).
6. Add **Outbound rule** or attach **NAT Gateway** (preferred for outbound SNAT scale).

### Create an Internal Load Balancer

Portal summary:
1. Choose **Internal** with **Frontend IP** on a subnet (private IP static recommended).
2. Backend pool: NICs/VMSS in same VNet (peering allowed when permitted).
3. Health probe configuration.
4. Load balancing rule (e.g., 443 → 443 for internal services).

### PowerShell and CLI Examples (Load Balancer)

```powershell
# Variables
$rg="myRG"; $loc="eastus"

# Public IP
$pip = New-AzPublicIpAddress -ResourceGroupName $rg -Location $loc -Name "lb-pip" -AllocationMethod Static -Sku Standard

# Frontend config
$fe = New-AzLoadBalancerFrontendIpConfig -Name "fe" -PublicIpAddress $pip

# Backend pool
$be = New-AzLoadBalancerBackendAddressPoolConfig -Name "bePool"

# Health probe (HTTP)
$probe = New-AzLoadBalancerProbeConfig -Name "httpProbe" -Protocol Http -Port 80 -RequestPath "/healthz" -IntervalInSeconds 5 -ProbeCount 2

# LB rule
$lbrule = New-AzLoadBalancerRuleConfig -Name "httpRule" -FrontendIpConfiguration $fe -BackendAddressPool $be -Probe $probe -Protocol Tcp -FrontendPort 80 -BackendPort 80 -EnableTcpReset -IdleTimeoutInMinutes 4

# Create Load Balancer
New-AzLoadBalancer -ResourceGroupName $rg -Name "myPublicLB" -Location $loc -FrontendIpConfiguration $fe -BackendAddressPool $be -LoadBalancingRule $lbrule -Probe $probe
```

```bash
# Azure CLI: Public Standard Load Balancer
rg=myRG
loc=eastus

# Public IP
az network public-ip create --resource-group $rg --name lb-pip --sku Standard --allocation-method static

# Load balancer
az network lb create \
	--resource-group $rg \
	--name myPublicLB \
	--sku Standard \
	--public-ip-address lb-pip \
	--frontend-ip-name fe \
	--backend-pool-name bePool

# Health probe
az network lb probe create \
	--resource-group $rg \
	--lb-name myPublicLB \
	--name httpProbe \
	--protocol http \
	--port 80 \
	--path /healthz

# LB rule
az network lb rule create \
	--resource-group $rg \
	--lb-name myPublicLB \
	--name httpRule \
	--protocol tcp \
	--frontend-port 80 \
	--backend-port 80 \
	--frontend-ip-name fe \
	--backend-pool-name bePool \
	--probe-name httpProbe \
	--enable-tcp-reset true \
	--idle-timeout 4
```

---

## Application Gateway (Layer 7)

### When to Use Application Gateway

- HTTP/HTTPS workloads needing path- or host-based routing
- TLS termination/offload, end-to-end TLS, or mutual TLS
- Web Application Firewall (OWASP CRS) protection
- URL rewrite, header manipulation, session affinity (cookie-based), connection draining
- Autoscaling and zone redundancy

### SKU Overview

| SKU | Features | Notes |
|-----|----------|-------|
| **Standard v2** | Autoscale, zone redundant, HTTP/2, WebSocket, rewrite rules | Production-ready |
| **WAF v2** | Standard v2 + WAF (detection/prevention), managed rules, custom rules | Recommended for internet-facing apps |

> Use v2 SKUs for all new deployments.

### Key Concepts

- **Frontend IP**: Public or private; bound to listeners.
- **Listeners**: Bind frontend + protocol + port + hostnames; basic or multi-site.
- **HTTP settings**: Backend port, protocol, TLS options, cookie affinity, connection draining, host override.
- **Backend pool**: IPs/FQDNs/NICs/VMSS/App Service; must allow probe source.
- **Health probes**: Custom path, interval, timeout, unhealthy threshold.
- **Routing rules**: Map listener → backend pool via HTTP settings; basic or path-based.
- **Rewrite rules**: Modify headers/URL at request/response stages.
- **Autoscale**: Min/max instances (v2); zone redundant optional.

### Web Application Firewall (WAF)

- Modes: **Detection** (log only) or **Prevention** (block).
- Rule Set: OWASP CRS (e.g., 3.2) with managed rules.
- Custom rules: IP match, geolocation, size constraints, methods, rate limiting.
- Exclusions: Fields to skip inspection (e.g., headers, JSON body elements).
- Logging: Send to Log Analytics, Storage, or Event Hub.

### Create an Application Gateway

Prerequisites: Dedicated subnet (only App Gateway), Standard v2/WAF v2 SKU, public or private frontend IP, backend reachable, NSGs permit probe traffic.

Portal summary:
1. Create VNet with dedicated **subnet** (e.g., `AppGwSubnet`).
2. Create **Application Gateway** → SKU: Standard v2 or WAF v2; set autoscale and zones if needed.
3. Configure **Frontend IP** (public, private, or both).
4. Add **Listeners** (basic or multi-site with host headers).
5. Configure **Backend pools** (IP/FQDN/VMSS/App Service).
6. Configure **HTTP settings** (backend port, protocol, cookie affinity, probe path, host override, TLS certs).
7. Add **Health probe** (path, interval, timeout, threshold).
8. Create **Routing rule** (basic or path-based) mapping listener → backend + HTTP settings.
9. (Optional) Enable **WAF** in prevention mode and select CRS version.
10. Review + Create.

### PowerShell and CLI Examples (App Gateway)

```powershell
# Variables
$rg="myRG"; $loc="eastus"; $vnetName="agwVnet"; $subnetName="AppGwSubnet"

# VNet + subnet
$vnet = New-AzVirtualNetwork -ResourceGroupName $rg -Location $loc -Name $vnetName -AddressPrefix "10.10.0.0/16" -Subnet (New-AzVirtualNetworkSubnetConfig -Name $subnetName -AddressPrefix "10.10.1.0/24")

# Public IP
$pip = New-AzPublicIpAddress -ResourceGroupName $rg -Location $loc -Name "agw-pip" -Sku Standard -AllocationMethod Static

# Frontend IP config
$fip = New-AzApplicationGatewayFrontendIPConfig -Name "fe" -PublicIpAddress $pip

# Frontend port
$fp = New-AzApplicationGatewayFrontendPort -Name "fePort" -Port 80

# Backend pool
$be = New-AzApplicationGatewayBackendAddressPool -Name "bePool" -BackendIpAddress 10.20.1.4,10.20.1.5

# HTTP settings
$httpsettings = New-AzApplicationGatewayBackendHttpSetting -Name "httpSetting" -Port 80 -Protocol Http -CookieBasedAffinity Disabled -RequestTimeout 30

# Health probe
$probe = New-AzApplicationGatewayProbeConfig -Name "healthProbe" -Protocol Http -HostName "app.contoso.com" -Path "/healthz" -Interval 5 -Timeout 15 -UnhealthyThreshold 3

# Listener
$listener = New-AzApplicationGatewayHttpListener -Name "feListener" -FrontendIpConfiguration $fip -FrontendPort $fp -Protocol Http -HostName "app.contoso.com"

# Rule
$rule = New-AzApplicationGatewayRequestRoutingRule -Name "rule1" -RuleType Basic -HttpListener $listener -BackendAddressPool $be -BackendHttpSettings $httpsettings -Probe $probe

# SKU + autoscale
$sku = New-AzApplicationGatewaySku -Name "Standard_v2" -Tier "Standard_v2" -Capacity 2
$autoscale = New-AzApplicationGatewayAutoscaleConfiguration -MinCapacity 2 -MaxCapacity 10

# Create App Gateway
New-AzApplicationGateway -ResourceGroupName $rg -Location $loc -Name "myAppGw" -BackendAddressPools $be -BackendHttpSettingsCollection $httpsettings -FrontendIpConfigurations $fip -GatewayIPConfigurations (New-AzApplicationGatewayIPConfiguration -Name "gatewayConfig" -Subnet $vnet.Subnets[0]) -FrontendPorts $fp -HttpListeners $listener -RequestRoutingRules $rule -Sku $sku -AutoscaleConfiguration $autoscale -Probes $probe
```

```bash
# Azure CLI: Application Gateway (Standard v2)
rg=myRG
loc=eastus
vnet=agwVnet
subnet=AppGwSubnet
pip=agw-pip

# Create VNet and subnet
az network vnet create --resource-group $rg --name $vnet --address-prefix 10.10.0.0/16 --subnet-name $subnet --subnet-prefix 10.10.1.0/24

# Public IP
az network public-ip create --resource-group $rg --name $pip --sku Standard --allocation-method static

# Create App Gateway
az network application-gateway create \
	--resource-group $rg \
	--name myAppGw \
	--location $loc \
	--sku Standard_v2 \
	--capacity 2 \
	--vnet-name $vnet \
	--subnet $subnet \
	--frontend-port 80 \
	--http-settings-port 80 \
	--http-settings-protocol Http \
	--public-ip-address $pip \
	--servers 10.20.1.4 10.20.1.5 \
	--routing-rule-type Basic \
	--probe-path /healthz \
	--probe-interval 5 \
	--probe-timeout 15 \
	--probe-unhealthy-threshold 3
```

---

## Traffic Manager vs Front Door vs Gateway vs Load Balancer

- **Azure Load Balancer (L4)**: TCP/UDP, low latency, regional scope, no TLS termination, HA Ports supported.
- **Application Gateway (L7)**: HTTP/HTTPS reverse proxy, WAF, path/host routing, TLS offload, rewrite, autoscale.
- **Traffic Manager (DNS-based)**: Global DNS load balancing (priority, weighted, performance, geographic, multi-value, subnet). Not a proxy; client resolves endpoints.
- **Azure Front Door (anycast edge proxy)**: Global entry, L7 routing, WAF, caching/acceleration, TLS offload, good for global web apps.

---

## Design Decision Matrix

| Requirement | Use |
|-------------|-----|
| TCP/UDP only, low latency, regional | Azure Load Balancer |
| HTTP/HTTPS with path/host routing, WAF | Application Gateway (WAF v2) |
| Global failover/geo-routing, DNS-based | Traffic Manager |
| Global edge acceleration + WAF + CDN | Azure Front Door |
| Full port range to NVAs | Load Balancer with HA Ports |
| Outbound SNAT scale | NAT Gateway with Load Balancer |

---

## Best Practices

- Prefer **Standard SKUs** for Load Balancer and public IP; enable zone redundancy.
- **NSGs**: Standard LB frontends are closed by default—explicitly allow traffic.
- **Health probes**: Lightweight endpoints (`/healthz` returning 200); align app readiness with probe success.
- **Idle timeout/TCP reset**: Configure to match application expectations; enable TCP reset for faster recovery.
- **Stateless design**: Limit session affinity; use client IP affinity only when necessary.
- **NAT Gateway**: Offload SNAT from Load Balancer for large-scale outbound.
- **Dedicated subnet for App Gateway**; avoid mixing resources.
- **WAF**: Start in Detection, monitor, then switch to Prevention; tune exclusions.
- **Certificates**: Use Key Vault integration for TLS certs; automate renewal.
- **Autoscaling**: Set min/max for App Gateway v2; monitor capacity metrics.

---

## Exam Tips

- **LB SKUs**: Basic (open, no zones) vs Standard (secure by default, zones, HA Ports).
- **Health probes** determine backend availability; failure removes instance from rotation.
- **HA Ports**: Single rule for all TCP/UDP ports—common for NVAs.
- **Outbound**: Standard LB requires outbound rule or NAT gateway for SNAT.
- **NSGs with Standard LB**: Must allow inbound explicitly (default deny).
- **App Gateway subnet**: Dedicated subnet required; one App Gateway per subnet.
- **WAF modes**: Detection (log) vs Prevention (block); OWASP CRS version matters.
- **Traffic Manager**: DNS-based only; **Front Door** is a global proxy with WAF.
- **Session persistence**: LB supports 5-tuple/client IP affinity; App Gateway supports cookie-based affinity.

---

## Scenarios

### Scenario 1: Public Web App with WAF and Path Routing
- Requirement: Internet-facing web app; `/api` to API backend; `/` to web front-end; OWASP protection.
- Solution: Application Gateway **WAF v2** with two backend pools (web, api), path-based rule, WAF in prevention mode.

### Scenario 2: Internal Line-of-Business App
- Requirement: Internal-only HTTP app; no internet exposure.
- Solution: **Internal Load Balancer** (Standard) with private frontend; NSG allows required ports; optional NAT Gateway for outbound.

### Scenario 3: High-Throughput NVAs
- Requirement: Virtual appliance needs all ports balanced for east-west traffic.
- Solution: Standard Load Balancer with **HA Ports** rule and custom probe; backend NVAs in availability set/VMSS.

### Scenario 4: Global Failover Between Regions
- Requirement: Active/passive DR for web app.
- Solution: **Traffic Manager (priority)** for DNS failover or **Front Door** for global proxy; backends are regional App Gateways.

### Scenario 5: Per-VM Admin Access
- Requirement: RDP/SSH to individual VMs without broad exposure.
- Solution: Standard Load Balancer **Inbound NAT rules** mapping unique frontend ports to VM-specific ports; restrict via NSGs/Just-In-Time.

---

## Troubleshooting

### Load Balancer Issues

- Backend not receiving traffic: Check probe status; verify NSG/UDR allow probe source and data path.
- Port closed externally: Standard LB requires NSG allow on subnet/NIC; verify public IP firewall/NVA rules.
- SNAT exhaustion: Use NAT Gateway or outbound rules; reduce long-lived connections.
- Asymmetric routing: Ensure UDRs do not bypass LB on return path; maintain symmetric routing for NVAs.

### Application Gateway Issues

- 502/504 errors: Check health probes, HTTP settings port/protocol, host header override, TLS trust chain.
- WAF blocking valid traffic: Run Detection, review logs, add exclusions/custom rules, then switch to Prevention.
- Listener failures: Validate certificates for HTTPS listener and host names for multi-site listeners.
- Subnet/NSG: App Gateway subnet must allow outbound to backends; avoid UDRs that break probe path.

### Diagnostics and Tools

- **Network Watcher**: Connection troubleshoot, IP flow verify, NSG flow logs.
- **Metrics**: App Gateway (throughput, current connections, capacity units, healthy/unhealthy hosts); Load Balancer (data path availability, SNAT ports used, health probe status).
- **Logs**: App Gateway access/WAF logs; Load Balancer flow via NSG flow logs; Front Door/Traffic Manager diagnostics.

---

## Command Quick Reference

```powershell
# Load Balancer: add backend pool addresses (VMSS example)
$vmss = Get-AzVmss -ResourceGroupName $rg -VMScaleSetName "myVMSS"
Add-AzLoadBalancerBackendAddressPoolConfig -Name bePool -LoadBalancer $lb -BackendIPConfigurationId $vmss.VirtualMachineProfile.NetworkProfile.NetworkInterfaceConfigurations[0].IpConfigurations[0].Id

# Load Balancer: add inbound NAT rule
Add-AzLoadBalancerInboundNatRuleConfig -LoadBalancer $lb -Name "rdpVM1" -FrontendIpConfiguration $lb.FrontendIpConfigurations[0] -Protocol Tcp -FrontendPort 50001 -BackendPort 3389

# App Gateway: add path-based rule
Add-AzApplicationGatewayUrlPathMapConfig -Name "pathMap" -DefaultBackendAddressPool $beWeb -DefaultBackendHttpSettings $httpSettingWeb -PathRules \
	(New-AzApplicationGatewayPathRuleConfig -Name "api" -Paths "/api/*" -BackendAddressPool $beApi -BackendHttpSettings $httpSettingApi)

# App Gateway: enable WAF prevention
Set-AzApplicationGatewayWebApplicationFirewallConfiguration -ApplicationGateway $agw -Enabled $true -FirewallMode Prevention -RuleSetType OWASP -RuleSetVersion "3.2"
```

```bash
# CLI: Add backend to LB pool
az network lb address-pool address add --resource-group $rg --lb-name myPublicLB --pool-name bePool --vnet myVNet --ip-address 10.0.1.4

# CLI: Create inbound NAT rule
az network lb inbound-nat-rule create \
	--resource-group $rg --lb-name myPublicLB \
	--name rdpVM1 --protocol Tcp --frontend-port 50001 --backend-port 3389 --frontend-ip-name fe

# CLI: App Gateway path-based rule
az network application-gateway url-path-map create \
	--resource-group $rg --gateway-name myAppGw \
	--name pathMap1 --paths /api/* \
	--address-pool beApi --http-settings httpSettingApi \
	--default-address-pool beWeb --default-http-settings httpSettingWeb

# CLI: Enable WAF prevention
az network application-gateway waf-config set \
	--resource-group $rg --gateway-name myAppGw \
	--enabled true --firewall-mode Prevention \
	--rule-set-type OWASP --rule-set-version 3.2
```

---

**Quick Recap**: Use **Load Balancer Standard** for Layer 4 TCP/UDP with health probes, HA Ports, and outbound rules; use **Application Gateway v2** for Layer 7 routing, TLS offload, and WAF; use **Traffic Manager** for DNS-based global routing; use **Front Door** for global edge acceleration and WAF.
