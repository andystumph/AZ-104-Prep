# Monitor Resources with Azure Monitor

> **Exam Weight:** Monitoring and Backup 10-15% (monitoring is critical for operational excellence)

## Table of Contents
- [Overview](#overview)
- [Azure Monitor Components](#azure-monitor-components)
- [Metrics](#metrics)
  - [Metric Types](#metric-types)
  - [View and Analyze Metrics](#view-and-analyze-metrics)
  - [Metric Alerts](#metric-alerts)
- [Logs and Log Analytics](#logs-and-log-analytics)
  - [Log Analytics Workspace](#log-analytics-workspace)
  - [Kusto Query Language (KQL)](#kusto-query-language-kql)
  - [Common KQL Queries](#common-kql-queries)
  - [Log Alerts](#log-alerts)
- [Diagnostic Settings](#diagnostic-settings)
- [Application Insights](#application-insights)
- [Workbooks](#workbooks)
- [Action Groups](#action-groups)
- [Alert Processing Rules](#alert-processing-rules)
- [Network Monitoring](#network-monitoring)
- [VM Insights](#vm-insights)
- [Container Insights](#container-insights)
- [Best Practices](#best-practices)
- [PowerShell and CLI](#powershell-and-cli)
- [Exam Tips](#exam-tips)
- [Scenarios](#scenarios)
- [Troubleshooting](#troubleshooting)
- [Command Quick Reference](#command-quick-reference)

---

## Overview

**Azure Monitor** provides full-stack monitoring for applications and infrastructure across Azure, hybrid, and multi-cloud environments. It collects, analyzes, and acts on telemetry data through:
- **Metrics:** Numerical time-series data (CPU %, memory, network throughput)
- **Logs:** Event and diagnostic data stored in Log Analytics workspaces
- **Alerts:** Proactive notifications based on thresholds or patterns
- **Visualizations:** Dashboards, workbooks, and Metrics Explorer

---

## Azure Monitor Components

| Component | Purpose | Data Type |
|-----------|---------|-----------|
| **Metrics** | Near real-time numerical data | Time-series (1-min granularity) |
| **Logs** | Detailed event/diagnostic data | Semi-structured text/JSON |
| **Alerts** | Proactive notifications | Triggered by metrics or logs |
| **Action Groups** | Define response actions (email, webhook, runbook) | Alert destinations |
| **Workbooks** | Interactive reports combining metrics/logs | Visualization |
| **Application Insights** | APM for web apps | Traces, requests, exceptions |
| **VM Insights** | VM performance and dependencies | Agent-based monitoring |
| **Container Insights** | AKS/ACI monitoring | Container logs and metrics |

---

## Metrics

### Metric Types

| Type | Description | Retention | Examples |
|------|-------------|-----------|----------|
| **Platform Metrics** | Automatically collected from Azure resources | 93 days | CPU %, Disk IOPS, Network In/Out |
| **Guest OS Metrics** | Collected via Azure Monitor Agent (AMA) | 93 days | Memory %, Disk space, Process count |
| **Custom Metrics** | Application-specific metrics via API | 93 days | Order count, login failures |
| **Prometheus Metrics** | Scraped from containers/VMs | Configurable | Container CPU, HTTP requests |

### View and Analyze Metrics

**Portal:**
1. Navigate to resource → **Metrics**.
2. Select **Metric namespace** (e.g., Virtual Machine).
3. Choose **Metric** (e.g., Percentage CPU).
4. Set **Aggregation** (Avg, Max, Min, Sum, Count).
5. Add filters (e.g., by VM name) and splitting (e.g., by disk).
6. Save to dashboard or pin to Azure Dashboard.

**Aggregation Types:**
- **Average:** Mean value over time interval
- **Maximum:** Peak value (useful for spikes)
- **Minimum:** Lowest value
- **Sum:** Total (useful for counters like network bytes)
- **Count:** Number of data points

### Metric Alerts

**Alert Types:**
- **Static Threshold:** Fire when metric crosses fixed value (e.g., CPU > 80%)
- **Dynamic Threshold:** Use machine learning to detect anomalies
- **Metric Measurement:** Log query results treated as metric values

**Create Metric Alert:**
1. Navigate to **Monitor** → **Alerts** → **Create alert rule**.
2. Select **Scope** (resource or resource group).
3. Choose **Condition** (metric, threshold, aggregation).
4. Set **Threshold** (static or dynamic).
5. Configure **Action group** (notifications and actions).
6. Set **Alert details** (severity, name, description).
7. Review and create.

---

## Logs and Log Analytics

### Log Analytics Workspace

**Purpose:** Centralized repository for logs from Azure Monitor, VMs, containers, applications, and security tools.

**Key Features:**
- Kusto Query Language (KQL) for log analysis
- 30-730+ days retention (configurable by table)
- Role-based access control (RBAC)
- Integration with Azure Sentinel, Microsoft Defender

**Create Workspace:**

**Portal:**
1. Search **Log Analytics workspaces** → **Create**.
2. Select **Subscription**, **Resource group**, **Name**, **Region**.
3. Choose **Pricing tier** (Pay-as-you-go, Capacity Reservation).
4. Review and create.

**Common Tables:**
| Table | Purpose |
|-------|---------|
| **AzureActivity** | Subscription-level events (create, delete, start, stop) |
| **AzureDiagnostics** | Resource diagnostic logs |
| **Heartbeat** | VM agent health (every 5 minutes) |
| **Perf** | Performance counters (CPU, memory, disk) |
| **Syslog** | Linux system logs |
| **Event** | Windows event logs |
| **ContainerLog** | Container stdout/stderr |
| **KubeEvents** | Kubernetes events |

### Kusto Query Language (KQL)

**Basic Syntax:**
```kql
TableName
| where Condition
| project Column1, Column2
| summarize Count=count() by Column1
| order by Count desc
| take 10
```

**Common Operators:**
- `where`: Filter rows (e.g., `where TimeGenerated > ago(1h)`)
- `project`: Select columns (e.g., `project Computer, CounterName, CounterValue`)
- `summarize`: Aggregate (e.g., `summarize avg(CounterValue) by Computer`)
- `order by`: Sort results
- `take` / `limit`: Limit rows returned
- `join`: Combine tables
- `extend`: Create calculated columns

### Common KQL Queries

**VM CPU Usage (Last Hour):**
```kql
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| summarize AvgCPU = avg(CounterValue) by Computer
| order by AvgCPU desc
```

**Failed Sign-ins (Azure AD):**
```kql
SigninLogs
| where TimeGenerated > ago(24h)
| where ResultType != "0"
| summarize FailedCount = count() by UserPrincipalName, ResultDescription
| order by FailedCount desc
```

**NSG Flow Logs (Top Talkers):**
```kql
AzureNetworkAnalytics_CL
| where TimeGenerated > ago(1h)
| summarize TotalBytes = sum(FlowCount_d) by SrcIP_s, DestIP_s
| order by TotalBytes desc
| take 10
```

**VM Availability (Heartbeat):**
```kql
Heartbeat
| summarize LastHeartbeat = max(TimeGenerated) by Computer
| extend Status = iff(LastHeartbeat > ago(5m), "Online", "Offline")
| project Computer, Status, LastHeartbeat
```

**Container Restart Count:**
```kql
KubePodInventory
| where TimeGenerated > ago(24h)
| summarize RestartCount = sum(ContainerRestartCount) by ContainerName, Namespace
| order by RestartCount desc
```

### Log Alerts

**Create Log Alert:**
1. Navigate to **Monitor** → **Alerts** → **Create alert rule**.
2. Select **Scope** (Log Analytics workspace or resource).
3. Choose **Condition** → **Custom log search**.
4. Write **KQL query** (e.g., `Heartbeat | where TimeGenerated > ago(5m) | summarize count() by Computer | where count_ == 0`).
5. Set **Alert logic** (threshold, evaluation frequency).
6. Configure **Action group**.
7. Set **Alert details** and create.

---

## Diagnostic Settings

**Purpose:** Route resource logs and metrics to destinations (Log Analytics, Storage Account, Event Hub).

**Create Diagnostic Setting:**

**Portal:**
1. Navigate to resource → **Diagnostic settings** → **Add diagnostic setting**.
2. Enter **Name**.
3. Select **Logs** categories (e.g., Administrative, Security, ServiceHealth).
4. Select **Metrics** (AllMetrics for comprehensive collection).
5. Choose **Destination**:
   - **Log Analytics workspace** (for querying)
   - **Storage account** (long-term archive)
   - **Event Hub** (streaming to external systems)
6. Save.

**Common Log Categories:**
| Category | Description |
|----------|-------------|
| **Administrative** | Create, update, delete operations |
| **Security** | Authentication, authorization events |
| **ServiceHealth** | Service health notifications |
| **Alert** | Alert rule evaluations |
| **Recommendation** | Azure Advisor recommendations |
| **Policy** | Azure Policy compliance events |
| **Autoscale** | Autoscale operations |

---

## Application Insights

**Purpose:** Application Performance Management (APM) for web apps, APIs, and microservices.

**Key Features:**
- Request rates, response times, failure rates
- Dependency tracking (SQL, Redis, HTTP calls)
- Exception tracking with stack traces
- Live metrics stream
- Application map (visual topology)
- Smart detection (anomaly detection via ML)

**Enable Application Insights:**

**For App Service:**
1. Navigate to App Service → **Application Insights**.
2. Click **Turn on Application Insights**.
3. Select existing workspace or create new.
4. Choose instrumentation level (Recommended).
5. Apply and restart app.

**For VMs/Containers:**
- Install Application Insights SDK or agent
- Configure connection string or instrumentation key

**Common Queries:**

**Slowest Requests:**
```kql
requests
| where timestamp > ago(1h)
| summarize AvgDuration = avg(duration), Count = count() by name
| order by AvgDuration desc
| take 10
```

**Exception Rate:**
```kql
exceptions
| where timestamp > ago(24h)
| summarize ExceptionCount = count() by type, outerMessage
| order by ExceptionCount desc
```

---

## Workbooks

**Purpose:** Interactive reports combining metrics, logs, text, and parameters for custom dashboards.

**Create Workbook:**
1. Navigate to **Monitor** → **Workbooks** → **New**.
2. Add **Sections**:
   - **Text:** Markdown formatting
   - **Metrics:** Metrics Explorer charts
   - **Logs:** KQL query results
   - **Parameters:** Dropdown filters (subscription, resource group, time range)
3. Customize layout, styling, and visualizations.
4. Save and share (public gallery or private).

**Templates:**
- VM Insights Performance
- Failure Analysis
- Traffic Analytics
- Container Monitoring

---

## Action Groups

**Purpose:** Define notification and automation actions for alerts.

**Action Types:**
| Type | Use Case |
|------|----------|
| **Email/SMS/Push/Voice** | Human notification |
| **Webhook** | Trigger external HTTP endpoint |
| **Logic App** | Complex workflow automation |
| **Azure Function** | Custom code execution |
| **Automation Runbook** | PowerShell/Python scripts |
| **ITSM** | Create incidents in ServiceNow, BMC |
| **Secure Webhook** | Azure AD authenticated webhook |

**Create Action Group:**

**Portal:**
1. Navigate to **Monitor** → **Alerts** → **Action groups** → **Create**.
2. Select **Subscription**, **Resource group**, **Name**, **Display name**.
3. Add **Actions**:
   - Email: `admin@contoso.com`
   - SMS: `+1-555-1234`
   - Webhook: `https://contoso.com/alert-webhook`
4. Save.

---

## Alert Processing Rules

**Purpose:** Suppress or modify alert notifications during maintenance windows or for specific scopes.

**Use Cases:**
- Suppress alerts during planned maintenance
- Route alerts to different action groups based on severity
- Add custom properties to alert payload

**Create Processing Rule:**
1. Navigate to **Monitor** → **Alerts** → **Alert processing rules** → **Create**.
2. Select **Scope** (subscription, resource group, or specific resources).
3. Define **Filter** (severity, alert rule name, monitor service).
4. Choose **Rule type**:
   - **Suppress notifications**
   - **Apply action group**
5. Set **Schedule** (always, one-time, recurring).
6. Review and create.

---

## Network Monitoring

**Tools:**
- **Network Watcher:** Connection troubleshoot, NSG flow logs, packet capture
- **Connection Monitor:** End-to-end connectivity monitoring (latency, packet loss)
- **Traffic Analytics:** NSG flow log analysis with geo mapping
- **NPM (Network Performance Monitor):** Deprecated; use Connection Monitor v2

**Connection Monitor Example:**

**Create Connection Monitor:**
1. Navigate to **Network Watcher** → **Connection monitor** → **Create**.
2. Add **Source endpoints** (VMs, on-prem agents).
3. Add **Destination endpoints** (VMs, URLs, IPs).
4. Define **Test configuration** (protocol TCP/HTTP/ICMP, port, frequency).
5. Add to **Test group** and create.
6. View **Metrics** (reachability %, latency ms, checks failed).

---

## VM Insights

**Purpose:** Monitor VM performance, health, and dependencies with pre-built workbooks.

**Features:**
- Performance charts (CPU, memory, disk, network)
- Dependency mapping (inbound/outbound connections)
- Health monitoring (critical/warning/healthy)
- Log Analytics integration

**Enable VM Insights:**

**Portal:**
1. Navigate to **Monitor** → **Virtual Machines** → **Get started**.
2. Select **VM** → **Enable**.
3. Choose **Log Analytics workspace** (creates/uses Data Collection Rule).
4. Wait for agent deployment (Azure Monitor Agent).
5. View **Performance** and **Map** tabs.

**Data Collected:**
- Performance counters (CPU, memory, disk, network)
- Processes running on VM
- Network connections (dependency mapping)

---

## Container Insights

**Purpose:** Monitor AKS clusters and ACI containers with logs and metrics.

**Enable for AKS:**

**Portal:**
1. Navigate to AKS cluster → **Insights**.
2. Click **Configure monitoring**.
3. Select **Log Analytics workspace**.
4. Enable (deploys OMSAgent DaemonSet).

**Common Queries:**

**Pod Restarts:**
```kql
KubePodInventory
| where TimeGenerated > ago(24h)
| summarize RestartCount = sum(ContainerRestartCount) by Name, Namespace
| where RestartCount > 0
| order by RestartCount desc
```

**Node CPU Usage:**
```kql
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "K8SNode"
| where CounterName == "cpuUsageNanoCores"
| summarize AvgCPU = avg(CounterValue) by Computer
| order by AvgCPU desc
```

---

## Best Practices

- **Workspaces:** Use one workspace per environment (dev, staging, prod) or per region.
- **Retention:** Configure retention per table (Interactive 30-730 days, Archive up to 7 years).
- **Alerts:** Set appropriate severity (0=Critical, 1=Error, 2=Warning, 3=Informational).
- **Action Groups:** Create reusable action groups (e.g., "OnCall-Team", "Ops-Email").
- **Query Optimization:** Use `where` early, avoid `*` projections, use `summarize` instead of `distinct`.
- **Cost Management:** Use basic logs for high-volume low-value data; sample metrics for containers.
- **Diagnostic Settings:** Enable for all critical resources; send to Log Analytics for analysis.
- **Dashboards:** Pin key metrics to Azure Portal dashboard for at-a-glance monitoring.

---

## PowerShell and CLI

```powershell
# Create Log Analytics Workspace
$rg = "monitoring-rg"
$loc = "eastus"
$wsName = "prodWorkspace"

New-AzOperationalInsightsWorkspace -ResourceGroupName $rg -Name $wsName -Location $loc -Sku PerGB2018

# Enable Diagnostic Settings (Storage Account)
$storage = Get-AzStorageAccount -ResourceGroupName $rg -Name "mystorageacct"
$ws = Get-AzOperationalInsightsWorkspace -ResourceGroupName $rg -Name $wsName

Set-AzDiagnosticSetting -ResourceId $storage.Id -WorkspaceId $ws.ResourceId -Enabled $true -Category "StorageRead","StorageWrite","StorageDelete" -MetricCategory "Transaction"

# Create Metric Alert (VM CPU > 80%)
$vm = Get-AzVM -ResourceGroupName $rg -Name "myVM"
$actionGroup = Get-AzActionGroup -ResourceGroupName $rg -Name "ops-team"

$criteria = New-AzMetricAlertRuleV2Criteria -MetricName "Percentage CPU" -TimeAggregation Average -Operator GreaterThan -Threshold 80

Add-AzMetricAlertRuleV2 -ResourceGroupName $rg -Name "vm-cpu-high" -TargetResourceId $vm.Id -Condition $criteria -ActionGroupId $actionGroup.Id -Severity 2 -WindowSize (New-TimeSpan -Minutes 5) -Frequency (New-TimeSpan -Minutes 1)

# Query Log Analytics
$ws = Get-AzOperationalInsightsWorkspace -ResourceGroupName $rg -Name $wsName
$query = "Heartbeat | where TimeGenerated > ago(1h) | summarize count() by Computer"
$results = Invoke-AzOperationalInsightsQuery -WorkspaceId $ws.CustomerId -Query $query
$results.Results

# Create Action Group
$emailReceiver = New-AzActionGroupReceiver -Name "AdminEmail" -EmailReceiver -EmailAddress "admin@contoso.com"
$smsReceiver = New-AzActionGroupReceiver -Name "AdminSMS" -SmsReceiver -CountryCode "1" -PhoneNumber "5551234567"

Set-AzActionGroup -ResourceGroupName $rg -Name "ops-team" -ShortName "ops" -Receiver $emailReceiver, $smsReceiver

# Enable VM Insights
$vm = Get-AzVM -ResourceGroupName $rg -Name "myVM"
Set-AzVMExtension -ResourceGroupName $rg -VMName $vm.Name -Name "MicrosoftMonitoringAgent" -Publisher "Microsoft.EnterpriseCloud.Monitoring" -ExtensionType "MicrosoftMonitoringAgent" -TypeHandlerVersion "1.0" -Settings @{"workspaceId"=$ws.CustomerId} -ProtectedSettings @{"workspaceKey"=(Get-AzOperationalInsightsWorkspaceSharedKey -ResourceGroupName $rg -Name $wsName).PrimarySharedKey}
```

```bash
# Azure CLI: Create Log Analytics Workspace
rg=monitoring-rg
loc=eastus
ws=prodWorkspace

az monitor log-analytics workspace create --resource-group $rg --workspace-name $ws --location $loc --sku PerGB2018

# Enable Diagnostic Settings
storageId=$(az storage account show --resource-group $rg --name mystorageacct --query id -o tsv)
wsId=$(az monitor log-analytics workspace show --resource-group $rg --workspace-name $ws --query id -o tsv)

az monitor diagnostic-settings create \
  --resource $storageId \
  --name "storage-diagnostics" \
  --workspace $wsId \
  --logs '[{"category":"StorageRead","enabled":true},{"category":"StorageWrite","enabled":true}]' \
  --metrics '[{"category":"Transaction","enabled":true}]'

# Create Metric Alert
vmId=$(az vm show --resource-group $rg --name myVM --query id -o tsv)
agId=$(az monitor action-group show --resource-group $rg --name ops-team --query id -o tsv)

az monitor metrics alert create \
  --resource-group $rg \
  --name vm-cpu-high \
  --scopes $vmId \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action $agId \
  --severity 2

# Query Log Analytics
wsId=$(az monitor log-analytics workspace show --resource-group $rg --workspace-name $ws --query customerId -o tsv)
az monitor log-analytics query --workspace $wsId --analytics-query "Heartbeat | where TimeGenerated > ago(1h) | summarize count() by Computer"

# Create Action Group
az monitor action-group create \
  --resource-group $rg \
  --name ops-team \
  --short-name ops \
  --email-receiver name=AdminEmail email-address=admin@contoso.com \
  --sms-receiver name=AdminSMS country-code=1 phone-number=5551234567
```

---

## Exam Tips

- **Metrics vs Logs:** Metrics = real-time numerical (CPU, memory); Logs = detailed events (sign-ins, errors).
- **Retention:** Metrics 93 days; Logs 30-730+ days (configurable).
- **Diagnostic Settings:** Required to route resource logs to Log Analytics/Storage/Event Hub.
- **Action Groups:** Reusable; attached to multiple alert rules.
- **KQL:** Master basic operators (`where`, `summarize`, `project`, `ago()`).
- **VM Insights:** Requires Azure Monitor Agent (replaces Log Analytics Agent).
- **Application Insights:** Automatic for App Service; SDK/agent for VMs/containers.
- **Alert Severity:** 0=Critical, 1=Error, 2=Warning, 3=Informational, 4=Verbose.
- **Dynamic Thresholds:** ML-based anomaly detection; useful when normal baseline varies.
- **Workspace Design:** Multi-region = multiple workspaces; cross-region queries incur egress costs.

---

## Scenarios

### Scenario 1: High CPU Alert for Production VMs

- **Requirement:** Alert Ops team when any production VM exceeds 85% CPU for 5 minutes.
- **Solution:**
  1. Create **Action Group** with email/SMS to Ops team.
  2. Create **Metric Alert** on resource group scope (all VMs).
  3. Condition: `Percentage CPU > 85`, aggregation `Average`, window `5 minutes`.
  4. Attach action group; set severity 1 (Error).

### Scenario 2: Detect Failed Sign-ins to Azure Portal

- **Requirement:** Alert Security team when 10+ failed sign-ins occur in 15 minutes.
- **Solution:**
  1. Enable **Azure AD diagnostic settings** to Log Analytics.
  2. Create **Log Alert** with query:
     ```kql
     SigninLogs
     | where TimeGenerated > ago(15m)
     | where ResultType != "0"
     | summarize FailedCount = count() by UserPrincipalName
     | where FailedCount >= 10
     ```
  3. Attach action group for Security team; severity 1.

### Scenario 3: Monitor Application Response Time

- **Requirement:** Track web app response times; alert if P95 latency > 2 seconds.
- **Solution:**
  1. Enable **Application Insights** for App Service.
  2. Create **Metric Alert** on Application Insights resource.
  3. Condition: `Server response time` percentile 95 > 2000 ms, window 5 minutes.
  4. Attach action group; severity 2 (Warning).

### Scenario 4: Container Restart Notifications

- **Requirement:** Alert when AKS pod restarts more than 3 times in 1 hour.
- **Solution:**
  1. Enable **Container Insights** for AKS.
  2. Create **Log Alert** with query:
     ```kql
     KubePodInventory
     | where TimeGenerated > ago(1h)
     | summarize RestartCount = sum(ContainerRestartCount) by Name, Namespace
     | where RestartCount > 3
     ```
  3. Attach action group; severity 2.

### Scenario 5: Suppress Alerts During Maintenance

- **Requirement:** Disable all alerts for VM during 2-hour maintenance window.
- **Solution:**
  1. Create **Alert Processing Rule** scoped to VM.
  2. Rule type: **Suppress notifications**.
  3. Schedule: One-time, specify start and end time.
  4. Apply rule (alerts still fire but notifications suppressed).

---

## Troubleshooting

### Metrics Not Appearing

- **Causes:** Resource recently created (wait 3-5 minutes), incorrect metric namespace, no activity generating metrics.
- **Actions:** Verify resource is running; check metric definition; refresh Metrics Explorer; ensure diagnostic settings enabled for guest OS metrics.

### Log Analytics Query Returns No Results

- **Causes:** Data not ingested yet (5-10 min delay), incorrect table name, time range too narrow, RBAC insufficient.
- **Actions:** Verify diagnostic settings enabled; check ingestion logs (`Operation | where OperationCategory == "Ingestion"`); expand time range; confirm Log Analytics Reader role.

### Alert Not Firing

- **Causes:** Condition never met, action group misconfigured, alert rule disabled, suppression rule active.
- **Actions:** Check **Alert history** (fired alerts); test action group (Send test notification); verify threshold and aggregation; review alert processing rules.

### VM Insights Shows No Data

- **Causes:** Azure Monitor Agent not installed, workspace not linked, data collection rule missing.
- **Actions:** Verify agent installed (`az vm extension list`); check VM → Insights → Configure; validate DCR associated with VM; review agent logs in `/var/log/` (Linux) or Event Viewer (Windows).

---

## Command Quick Reference

```powershell
# Get all alerts in subscription
Get-AzAlert

# Get fired alerts for resource
$vm = Get-AzVM -ResourceGroupName "myRG" -Name "myVM"
Get-AzAlert -TargetResourceId $vm.Id | Where-Object {$_.Properties.StartDateTime -gt (Get-Date).AddHours(-24)}

# Disable alert rule
$alert = Get-AzMetricAlertRuleV2 -ResourceGroupName "myRG" -Name "vm-cpu-high"
Update-AzMetricAlertRuleV2 -ResourceGroupName "myRG" -Name "vm-cpu-high" -Enabled $false

# Export Log Analytics query results to CSV
$results = Invoke-AzOperationalInsightsQuery -WorkspaceId $ws.CustomerId -Query "Heartbeat | summarize count() by Computer"
$results.Results | Export-Csv -Path "C:\temp\heartbeat.csv" -NoTypeInformation

# Get diagnostic settings
Get-AzDiagnosticSetting -ResourceId $vm.Id
```

```bash
# List all alerts
az monitor metrics alert list --resource-group myRG

# Test action group
az monitor action-group test-notifications create --action-group ops-team --alert-type servicehealth

# Get alert history
az monitor metrics alert show --resource-group myRG --name vm-cpu-high --query "lastUpdatedTime"

# Run KQL query
az monitor log-analytics query --workspace $wsId --analytics-query "Perf | where CounterName == '% Processor Time' | summarize avg(CounterValue) by Computer" --timespan P1D
```

---

**Quick Recap**: Use **Azure Monitor** for full-stack observability; **Metrics** for real-time numerical data; **Logs** for detailed analysis with KQL; **Alerts** for proactive notifications; **Action Groups** for automated responses; **Application Insights** for APM; **VM/Container Insights** for infrastructure monitoring. Configure diagnostic settings for all critical resources and design reusable action groups for operational efficiency.
