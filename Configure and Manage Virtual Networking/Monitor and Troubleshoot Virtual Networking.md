# Monitor and Troubleshoot Virtual Networking

> **Exam Weight:** Networking 15-20% (monitoring and troubleshooting features are routinely tested)

## Table of Contents
- [Overview](#overview)
- [Enable Network Watcher](#enable-network-watcher)
- [Core Diagnostics Tools](#core-diagnostics-tools)
	- [IP Flow Verify](#ip-flow-verify)
	- [Next Hop](#next-hop)
	- [Effective Security Rules](#effective-security-rules)
	- [Connection Troubleshoot](#connection-troubleshoot)
	- [Packet Capture](#packet-capture)
	- [NSG Flow Logs](#nsg-flow-logs)
	- [Topology](#topology)
- [Connection Monitor (v2)](#connection-monitor-v2)
- [VPN Gateway and Site-to-Site Diagnostics](#vpn-gateway-and-site-to-site-diagnostics)
- [ExpressRoute Monitoring](#expressroute-monitoring)
- [Metrics, Logs, and Alerts](#metrics-logs-and-alerts)
- [Troubleshooting Playbooks](#troubleshooting-playbooks)
- [Exam Tips](#exam-tips)
- [Scenarios](#scenarios)
- [Troubleshooting Reference](#troubleshooting-reference)
- [Command Quick Reference](#command-quick-reference)

---

## Overview

Azure networking diagnostics center around **Network Watcher**. It provides packet-level captures, flow logs, effective NSG rules, connection testing, and routing validation. Complementary services include **Connection Monitor v2** for synthetic tests, VPN Gateway/ExpressRoute diagnostics for hybrid links, and platform metrics/alerts for proactive detection.

---

## Enable Network Watcher

Network Watcher must be enabled per region. Enabling creates a Network Watcher resource; only one per region is needed.

**PowerShell:**
```powershell
Enable-AzNetworkWatcher -Location "eastus" -Name "NetworkWatcher_eastus" -ResourceGroupName "NetworkWatcherRG"
```

**CLI:**
```bash
az network watcher configure --locations eastus --resource-group NetworkWatcherRG --enabled true
```

**Portal:** Network Watcher → Overview → Enable for region.

---

## Core Diagnostics Tools

### IP Flow Verify
- Tests whether traffic is allowed or denied by NSG for a specific 5-tuple (src/dst IP, port, protocol, direction).
- Useful for fast NSG validation when packets drop.

```powershell
Test-AzNetworkWatcherIPFlow -NetworkWatcherName "NetworkWatcher_eastus" -ResourceGroupName "NetworkWatcherRG" `
	-Direction Outbound -Protocol TCP -LocalPort 443 -RemotePort 443 -LocalIPAddress 10.0.1.4 -RemoteIPAddress 52.239.192.0 `
	-TargetResourceId "/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/webvm1"
```

```bash
az network watcher test-ip-flow \
	--location eastus \
	--resource-group NetworkWatcherRG \
	--direction Outbound \
	--protocol TCP \
	--local 10.0.1.4:443 \
	--remote 52.239.192.0:443 \
	--vm webvm1
```

### Next Hop
- Determines the next hop type (VNet, Internet, Virtual Appliance, VNet Peering, None) for a destination.
- Validates effective routing and UDR impact.

```powershell
Get-AzNetworkWatcherNextHop -NetworkWatcherName "NetworkWatcher_eastus" -ResourceGroupName "NetworkWatcherRG" `
	-TargetVirtualMachineId "/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/webvm1" `
	-SourceIPAddress 10.0.1.4 -DestinationIPAddress 8.8.8.8
```

```bash
az network watcher show-next-hop \
	--location eastus \
	--resource-group NetworkWatcherRG \
	--source-ip 10.0.1.4 \
	--dest-ip 8.8.8.8 \
	--vm webvm1
```

### Effective Security Rules
- Shows combined NSG rules applied to a NIC (subnet + NIC NSGs).
- Quickly reveals which rule allows/denies traffic.

```powershell
Get-AzEffectiveNetworkSecurityGroup -NetworkInterfaceName "webvm1-nic" -ResourceGroupName "myRG"
```

```bash
az network nic list-effective-nsg --resource-group myRG --name webvm1-nic
```

### Connection Troubleshoot
- On-demand TCP/ICMP test from source to destination (VM, FQDN, IP) with hop-by-hop details.
- Helps isolate NSG, DNS, routing, or application port issues.

```powershell
Test-AzNetworkWatcherConnectivity -NetworkWatcherName "NetworkWatcher_eastus" -ResourceGroupName "NetworkWatcherRG" `
	-SourceId "/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/webvm1" `
	-DestinationAddress "10.0.2.4" -DestinationPort 443
```

```bash
az network watcher connection-monitor test-configuration add --help  # shows parameters
az network watcher test-connectivity \
	--source-resource webvm1 \
	--dest-address 10.0.2.4 \
	--dest-port 443 \
	--protocol Tcp \
	--port 443 \
	--resource-group myRG
```

### Packet Capture
- Captures packets on a VM NIC for deep inspection. Requires storage account or local file on VM.
- Great for intermittent issues and protocol analysis; keep duration/filters minimal to reduce cost.

```powershell
New-AzNetworkWatcherPacketCapture -NetworkWatcherName "NetworkWatcher_eastus" -ResourceGroupName "NetworkWatcherRG" `
	-TargetVirtualMachineId "/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/webvm1" `
	-PacketCaptureName "pcap1" -StorageAccountId "/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystg" `
	-TimeLimitInSeconds 600 -Filters @(New-AzPacketCaptureFilterConfig -Protocol Tcp -LocalPort 443)
```

```bash
az network watcher packet-capture create \
	--location eastus \
	--name pcap1 \
	--vm webvm1 \
	--storage-account mystg \
	--time-limit 600 \
	--protocol Tcp \
	--local-port 443
```

### NSG Flow Logs
- Flow-level logging (allow/deny) from NSGs to a storage account; supports Traffic Analytics.
- Use for auditing, security investigations, and flow pattern analysis.

```bash
# Enable NSG flow logs v2
az network watcher flow-log configure \
	--resource-group NetworkWatcherRG \
	--nsg myNSG \
	--enabled true \
	--retention 30 \
	--log-version 2 \
	--storage-account mystg

# Enable Traffic Analytics (optional)
az network watcher flow-log configure \
	--resource-group NetworkWatcherRG \
	--nsg myNSG \
	--enabled true \
	--retention 30 \
	--log-version 2 \
	--storage-account mystg \
	--traffic-analytics true \
	--workspace myLAW \
	--workspace-resource-group log-rg \
	--workspace-subscription <sub> \
	--workspace-region eastus
```

### Topology
- Visualizes resources and relationships in a VNet.
- Portal: Network Watcher → Topology → select subscription/resource group/vnet.

---

## Connection Monitor (v2)

Connection Monitor v2 provides continuous synthetic tests across regions and hybrid endpoints.

- **Sources**: VMs, Azure Arc servers, on-prem endpoints.
- **Destinations**: IP/FQDN/URL, ports, or other endpoints.
- **Protocols**: TCP, ICMP, HTTP/HTTPS.
- **Outputs**: Latency, packet loss, reachability, hop-by-hop path.
- **Alerts**: Create Azure Monitor alerts on metrics.

```bash
# Create a connection monitor v2
az network watcher connection-monitor create \
	--location eastus \
	--name cm-web-to-api \
	--resource-group NetworkWatcherRG \
	--source-resource webvm1 \
	--dest-address api.contoso.com \
	--dest-port 443 \
	--protocol Tcp \
	--test-frequency 300
```

Best practices:
- Group related tests; avoid excessive frequency to limit cost.
- Use tags/naming to map to app components.
- Alert on reachability and latency thresholds.

---

## VPN Gateway and Site-to-Site Diagnostics

- **VPN Gateway Diagnostic Logs**: Capture `GatewayDiagnosticLog`, `TunnelDiagnosticLog`, `RouteDiagnosticLog` to Log Analytics/Storage/Event Hub.
- **Connection Status**: Portal → VPN connections → view data in/out, status, and shared key; CLI/Pwsh equivalents below.
- **Packet Capture on Gateway**: Supported for IKE/IPsec debugging (run carefully; may be transient).
- **Common Issues**: Mismatched IKE/IPsec policy, pre-shared key mismatch, dead peer detection timers, fragmented packets (MTU), route propagation missing.

```bash
# Show VPN connection status
az network vpn-connection show --resource-group myRG --name hub-to-onprem --query "{status:connectionStatus, ingress:ingressBytesTransferred, egress:egressBytesTransferred}"

# Start gateway packet capture (short duration recommended)
az network watcher packet-capture create \
	--location eastus \
	--name gw-capture \
	--target-type Gateway \
	--target /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworkGateways/myVpnGw \
	--storage-account mystg \
	--time-limit 300

# Enable VPN gateway diagnostics to Log Analytics
az monitor diagnostic-settings create \
	--resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworkGateways/myVpnGw \
	--name gw-diag \
	--workspace /subscriptions/<sub>/resourceGroups/log-rg/providers/Microsoft.OperationalInsights/workspaces/myLAW \
	--logs '[{"category":"GatewayDiagnosticLog","enabled":true},{"category":"TunnelDiagnosticLog","enabled":true},{"category":"RouteDiagnosticLog","enabled":true}]'
```

---

## ExpressRoute Monitoring

- **Metrics**: Bits in/out, BGP availability, ARP availability, route counts.
- **NPM (Connection Monitor) + Resource Health**: Track circuit health and latency if using ExpressRoute + FastPath.
- **Diagnostic Settings**: Send `PeeringRouteLog`, `BGPRouteLog`, `ArpAvailabilityLog` to Log Analytics/Storage/Event Hub.

```bash
az monitor diagnostic-settings create \
	--resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/expressRouteCircuits/myCircuit \
	--name er-diag \
	--workspace /subscriptions/<sub>/resourceGroups/log-rg/providers/Microsoft.OperationalInsights/workspaces/myLAW \
	--logs '[{"category":"PeeringRouteLog","enabled":true},{"category":"BGPRouteLog","enabled":true},{"category":"ArpAvailabilityLog","enabled":true}]'
```

Best practices:
- Monitor BGP session state and route table size.
- Validate primary/secondary paths; ensure redundant peering up.
- Enable alerts on circuit provider status and utilization.

---

## Metrics, Logs, and Alerts

- **Metrics**: Throughput, data path availability (Load Balancer), SNAT port usage, connections, health probe status, App Gateway metrics if applicable.
- **Logs**: NSG flow logs, Activity Log, resource diagnostic logs (VPN, ER, App Gateway, Load Balancer), Connection Monitor results.
- **Alerts**: Azure Monitor metric alerts (latency, packet loss, probe failures), log alerts (NSG denies surge, VPN tunnel down), Activity Log alerts (route table changes).

**Example metric alert (CLI):**
```bash
az monitor metrics alert create \
	--name "lb-probe-fail" \
	--resource-group myRG \
	--scopes /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/loadBalancers/myPublicLB \
	--condition "avg LoadBalancerProbeHealth < 1" \
	--window-size 5m \
	--evaluation-frequency 5m \
	--action-group myAg
```

**Example log alert for VPN tunnel down (KQL):**
```kusto
AzureDiagnostics
| where Category == "TunnelDiagnosticLog"
| where connectionStatus_s == "Disconnected"
| summarize count() by connectionName_s, bin(TimeGenerated, 5m)
```

---

## Troubleshooting Playbooks

### VM-to-VM Connectivity (same VNet)
1. Check NSG effective rules on both NICs.
2. IP Flow Verify for allow/deny.
3. Connection Troubleshoot between NICs.
4. Verify custom routes (UDR) and next hop.
5. If still failing, Packet Capture on source/destination.

### Cross-VNet or Peered Networks
1. Confirm peering state (connected, allow forwarded traffic if NVAs involved).
2. Check UDRs for asymmetric routing.
3. Next Hop for destination IP to validate path.
4. IP Flow Verify and NSG rules.
5. Connection Troubleshoot; if appliance present, ensure return path via same appliance.

### Internet Egress Issues
1. Check default route and next hop (NAT Gateway, firewall, internet via LB outbound rule).
2. SNAT exhaustion? Review NAT Gateway/LB SNAT metrics.
3. IP Flow Verify outbound to target IP/port.
4. NSG flow logs to see denies.

### VPN Tunnel Down
1. Verify shared key and IKE/IPsec policy match.
2. Check on-prem device logs and Azure `TunnelDiagnosticLog`.
3. Ensure routes advertised/learned; check BGP if used.
4. MTU/fragmentation: lower MSS if needed.

### Application Gateway or LB Probe Failures
1. Confirm backend responds on probe path/port.
2. NSG/UDR not blocking probe source.
3. Check health probe metrics; use Connection Troubleshoot from probe subnet if needed.
4. Review TLS settings/cert chain for HTTPS probes.

---

## Exam Tips

- **Network Watcher scope**: Enable per region; required for diagnostics.
- **IP Flow Verify**: Fast allow/deny check against NSG rules.
- **Next Hop**: Validates UDR/routing decisions.
- **NSG Flow Logs**: Stored in Storage; Traffic Analytics for insights.
- **Connection Monitor v2**: Continuous tests; v1 is legacy.
- **Probe health** drives Load Balancer/App Gateway traffic decisions.
- **VPN diagnostics**: Gateway/Tunnel/Route logs; packet capture available on gateway.
- **ExpressRoute logs**: BGP/ARP/Route logs via diagnostic settings.
- **App Gateway subnet**: Dedicated; UDR/NSG must allow probes.

---

## Scenarios

### Scenario 1: Intermittent API Timeouts
- Use Connection Monitor v2 between web and API.
- Enable NSG flow logs to see drops; check SNAT exhaustion metrics.
- Capture packets during incident; validate next hop and UDRs.

### Scenario 2: VPN Users Report Drops
- Review TunnelDiagnosticLog for disconnects; alert on disconnect events.
- Check MTU/MSS; enable packet capture for IKE/IPsec.
- Validate on-prem device logs; align IKE/IPsec policies.

### Scenario 3: New NSG Blocks Production
- Run IP Flow Verify for failing tuple.
- Inspect Effective Security Rules on NIC to find the blocking rule.
- Review Activity Log for recent NSG changes; implement approval workflow.

### Scenario 4: App Gateway 502 Errors
- Check health probe response codes; confirm host header override.
- Validate backend TLS cert chain if HTTPS.
- Use Connection Troubleshoot from App Gateway subnet to backend.

### Scenario 5: Peered VNets with Appliance
- Ensure peering allows forwarded traffic and gateway transit if needed.
- Verify UDR symmetry (both directions through appliance).
- Use Next Hop and Connection Troubleshoot across VNets.

---

## Troubleshooting Reference

| Symptom | Likely Cause | Tool/Action |
|---------|--------------|-------------|
| Traffic denied unexpectedly | NSG rule priority or missing allow | IP Flow Verify; Effective Security Rules |
| Wrong path taken | UDR forcing different next hop | Next Hop; route table review |
| Probe failures on LB/App GW | Backend not listening, NSG/UDR blocking, TLS mismatch | Health probe logs; Connection Troubleshoot; NSG flow logs |
| VPN tunnel flaps | Key mismatch, policy mismatch, MTU issues | TunnelDiagnosticLog; gateway packet capture; device logs |
| High latency spikes | Congestion, routing detour, SNAT exhaustion | Connection Monitor metrics; LB SNAT metrics; Next Hop |
| DNS failures | Wrong resolver or blocked port 53 | Connection Troubleshoot to DNS IP; Packet Capture |

---

## Command Quick Reference

```powershell
# Enable Network Watcher in a region
Enable-AzNetworkWatcher -Location "eastus" -Name "NetworkWatcher_eastus" -ResourceGroupName "NetworkWatcherRG"

# IP Flow Verify
Test-AzNetworkWatcherIPFlow -NetworkWatcherName "NetworkWatcher_eastus" -ResourceGroupName "NetworkWatcherRG" -Direction Inbound -Protocol Tcp -LocalPort 443 -RemotePort 443 -LocalIPAddress 10.0.1.4 -RemoteIPAddress 52.239.192.0 -TargetResourceId "/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/webvm1"

# Next Hop
Get-AzNetworkWatcherNextHop -NetworkWatcherName "NetworkWatcher_eastus" -ResourceGroupName "NetworkWatcherRG" -TargetVirtualMachineId "/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/webvm1" -SourceIPAddress 10.0.1.4 -DestinationIPAddress 8.8.8.8

# Effective NSG rules
Get-AzEffectiveNetworkSecurityGroup -NetworkInterfaceName "webvm1-nic" -ResourceGroupName "myRG"

# Connection troubleshoot
Test-AzNetworkWatcherConnectivity -NetworkWatcherName "NetworkWatcher_eastus" -ResourceGroupName "NetworkWatcherRG" -SourceId "/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/webvm1" -DestinationAddress "10.0.2.4" -DestinationPort 443

# Packet capture
New-AzNetworkWatcherPacketCapture -NetworkWatcherName "NetworkWatcher_eastus" -ResourceGroupName "NetworkWatcherRG" -TargetVirtualMachineId "/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/webvm1" -PacketCaptureName "pcap1" -StorageAccountId "/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystg" -TimeLimitInSeconds 600

# Enable NSG flow logs
Set-AzNetworkWatcherConfigFlowLog -NetworkWatcherName "NetworkWatcher_eastus" -ResourceGroupName "NetworkWatcherRG" -TargetResourceId "/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/networkSecurityGroups/myNSG" -Enabled $true -RetentionInDays 30 -StorageId "/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystg" -FormatVersion 2

# VPN connection status
Get-AzVirtualNetworkGatewayConnection -Name "hub-to-onprem" -ResourceGroupName "myRG" | Select-Object Name, ConnectionStatus, EgressBytesTransferred, IngressBytesTransferred
```

```bash
# Enable Network Watcher
az network watcher configure --locations eastus --resource-group NetworkWatcherRG --enabled true

# IP Flow Verify
az network watcher test-ip-flow --location eastus --resource-group NetworkWatcherRG --direction Outbound --protocol TCP --local 10.0.1.4:443 --remote 52.239.192.0:443 --vm webvm1

# Next Hop
az network watcher show-next-hop --location eastus --resource-group NetworkWatcherRG --source-ip 10.0.1.4 --dest-ip 8.8.8.8 --vm webvm1

# Effective NSG rules
az network nic list-effective-nsg --resource-group myRG --name webvm1-nic

# Connection troubleshoot
az network watcher test-connectivity --source-resource webvm1 --dest-address 10.0.2.4 --dest-port 443 --protocol Tcp --resource-group myRG

# Packet capture
az network watcher packet-capture create --location eastus --name pcap1 --vm webvm1 --storage-account mystg --time-limit 600 --protocol Tcp --local-port 443

# Enable NSG flow logs v2
az network watcher flow-log configure --resource-group NetworkWatcherRG --nsg myNSG --enabled true --retention 30 --log-version 2 --storage-account mystg

# VPN connection status
az network vpn-connection show --resource-group myRG --name hub-to-onprem --query "{status:connectionStatus, ingress:ingressBytesTransferred, egress:egressBytesTransferred}"
```

---

**Quick Recap**: Enable Network Watcher per region. Use IP Flow Verify and Effective NSG to validate access, Next Hop to confirm routing, Connection Troubleshoot/Monitor for reachability and latency, NSG flow logs for allow/deny auditing, and diagnostic settings for VPN/ExpressRoute. Probe health drives load-balancing decisions; always check NSG/UDR symmetry.
