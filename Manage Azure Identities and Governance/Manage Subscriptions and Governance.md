# Manage Subscriptions and Governance

## Table of Contents
1. [Overview](#overview)
2. [Azure Policy](#azure-policy)
3. [Resource Locks](#resource-locks)
4. [Resource Tags](#resource-tags)
5. [Management Groups](#management-groups)
6. [Subscription Management](#subscription-management)
7. [Cost Management](#cost-management)
8. [Governance Best Practices](#governance-best-practices)
9. [PowerShell & CLI Commands](#powershell--cli-commands)
10. [Exam Tips](#exam-tips)
11. [Common Scenarios & Troubleshooting](#common-scenarios--troubleshooting)
12. [Related Topics](#related-topics)

---

## Overview

Azure governance provides tools to organize, control, and optimize your Azure resources at scale. This includes policy enforcement, cost control, resource organization, and compliance management across subscriptions and management groups.

### Key Governance Components

| Component | Purpose | Scope |
|-----------|---------|-------|
| **Azure Policy** | Enforce organizational standards and compliance | Management Group, Subscription, Resource Group |
| **Resource Locks** | Prevent accidental deletion or modification | Subscription, Resource Group, Resource |
| **Resource Tags** | Organize and categorize resources | Resource, Resource Group |
| **Management Groups** | Hierarchical organization of subscriptions | Tenant level |
| **Cost Management** | Monitor, allocate, and optimize spending | All scopes |
| **Blueprints** | Repeatable governance patterns | Management Group, Subscription |

### Azure Resource Hierarchy

```
Tenant (Root Management Group)
└── Management Groups
    └── Subscriptions
        └── Resource Groups
            └── Resources
```

**Exam Insight:** Understanding scope inheritance is critical. Policies and locks applied at higher scopes automatically inherit to child scopes. Tags do NOT inherit by default (but Cost Management can enable tag inheritance for reporting).

---

## Azure Policy

Azure Policy helps enforce organizational standards and assess compliance at scale by evaluating resource properties against business rules.

### Policy Core Concepts

#### Policy Definition
A business rule expressed in JSON format that evaluates resource properties.

**Built-in Policy Example:** "Allowed locations"
- Restricts which regions resources can be deployed to
- Effect: Deny deployment to non-compliant locations

#### Initiative Definition (Policy Set)
A collection of policy definitions grouped together to achieve a specific goal.

**Example:** "Enable Monitoring in Azure Security Center"
- Contains 100+ policies
- Monitors VMs, networks, SQL, and more

#### Policy Assignment
Assigning a policy or initiative to a specific scope (management group, subscription, or resource group).

### Policy Effects

| Effect | Behavior | Common Use Case |
|--------|----------|-----------------|
| **Deny** | Prevents non-compliant resource creation/update | Enforce allowed VM SKUs, regions, or resource types |
| **Audit** | Creates warning in activity log, doesn't block | Identify resources missing tags or encryption |
| **AuditIfNotExists** | Audits if related resource doesn't exist | Check if VMs have backup enabled |
| **DeployIfNotExists** | Automatically deploys related resource | Auto-enable diagnostic settings or VM extensions |
| **Modify** | Adds, updates, or removes tags or properties | Enforce required tags on resources |
| **Append** | Adds fields to resource during creation/update | Add subnet to virtual network |
| **Disabled** | Policy exists but doesn't evaluate | Temporarily disable enforcement |
| **Manual** | Requires manual attestation of compliance | Custom compliance scenarios |

**Exam Tip:** Know the difference between Audit (logs only) vs. Deny (blocks action) vs. DeployIfNotExists (remediates automatically).

### Compliance States

| State | Description |
|-------|-------------|
| **Compliant** | Resource meets policy requirements |
| **Non-Compliant** | Resource violates policy, identified for remediation |
| **Conflicting** | Multiple policies apply with contradictory requirements |
| **Not Started** | Policy evaluation hasn't begun |
| **Exempt** | Resource explicitly exempted from policy |

### Policy Evaluation Triggers

- Resource is created or updated
- Policy or initiative is newly assigned
- Policy or initiative is updated
- Standard compliance evaluation cycle (every 24 hours)

### Creating a Custom Policy Definition

**Portal Steps:**
1. Navigate to **Policy** → **Definitions**
2. Select **+ Policy definition**
3. Define **Definition location** (management group or subscription)
4. Enter **Name**, **Description**, and **Category**
5. Write policy rule in JSON (or use existing template)
6. Define **Parameters** (optional)
7. Click **Save**

**Custom Policy JSON Structure:**
```json
{
  "properties": {
    "displayName": "Require tag on resources",
    "policyType": "Custom",
    "mode": "Indexed",
    "description": "Enforces a required tag on resources",
    "metadata": {
      "category": "Tags"
    },
    "parameters": {
      "tagName": {
        "type": "String",
        "metadata": {
          "displayName": "Tag Name",
          "description": "Name of the tag, such as 'environment'"
        }
      }
    },
    "policyRule": {
      "if": {
        "field": "[concat('tags[', parameters('tagName'), ']')]",
        "exists": "false"
      },
      "then": {
        "effect": "deny"
      }
    }
  }
}
```

### Policy Assignment Process

**Portal Steps:**
1. Navigate to **Policy** → **Assignments**
2. Select **Assign policy** or **Assign initiative**
3. Configure:
   - **Scope:** Select subscription/resource group (with exclusions if needed)
   - **Basics:** Choose policy/initiative, provide assignment name
   - **Parameters:** Set parameter values (e.g., allowed locations)
   - **Remediation:** Enable managed identity if using DeployIfNotExists/Modify
   - **Non-compliance messages:** Custom message for denied requests
4. Click **Review + create**

**Key Configuration Options:**
- **Exclusions:** Specific resource groups or resources to skip
- **Policy Enforcement:** Enabled or Disabled (audit mode)
- **Managed Identity:** Required for policies that deploy or modify resources
- **Assignment Location:** Region for managed identity

### Policy Remediation

For existing non-compliant resources with DeployIfNotExists or Modify policies:

1. Navigate to **Policy** → **Remediation**
2. Select the policy assignment
3. Click **Create remediation task**
4. Select:
   - **Scope:** Which resources to remediate
   - **Re-evaluate resource compliance:** Before remediation
   - **Failure handling:** Stop or continue on errors
5. Click **Remediate**

**Important:** Remediation tasks use managed identities with proper RBAC roles.

### Policy Exemptions

Temporary or permanent exceptions to policy assignments.

**Exemption Categories:**
- **Waiver:** Resource doesn't need to meet requirement (permanent exception)
- **Mitigated:** Requirement met through alternative means (documented mitigation)

**Creating an Exemption:**
1. Navigate to **Policy** → **Exemptions**
2. Select **+ Add exemption**
3. Configure:
   - **Exemption scope:** Subscription, resource group, or resource
   - **Policy assignment:** Which policy to exempt
   - **Exemption category:** Waiver or Mitigated
   - **Expires on:** Optional expiration date
   - **Description:** Required justification
4. Click **Create**

---

## Resource Locks

Resource locks prevent accidental deletion or modification of critical Azure resources. They override RBAC permissions.

### Lock Types

| Lock Type | Read | Update | Delete |
|-----------|------|--------|--------|
| **ReadOnly** | ✓ | ✗ | ✗ |
| **CanNotDelete** | ✓ | ✓ | ✗ |

**Exam Scenario:** A user with Owner role CANNOT delete a resource if a CanNotDelete lock is applied. Locks override RBAC.

### Lock Behavior

- **Inheritance:** Locks applied at parent scope (subscription/resource group) inherit to all child resources
- **RBAC Override:** Even Owners must remove lock before performing blocked operations
- **Management Operations:** Locks prevent DELETE and POST operations, not management plane reads
- **Specific Resource Impacts:**
  - Storage account ReadOnly lock prevents listing keys
  - VM ReadOnly lock prevents start/stop operations
  - App Service ReadOnly lock prevents configuration changes

### Applying Locks via Portal

1. Navigate to the resource, resource group, or subscription
2. Select **Settings** → **Locks**
3. Click **+ Add**
4. Configure:
   - **Lock name:** Descriptive name
   - **Lock type:** Read-only or Delete
   - **Notes:** Reason for lock
5. Click **OK**

### Removing Locks

1. Navigate to the locked resource
2. Select **Settings** → **Locks**
3. Find the lock
4. Click **Delete** (trash icon)

**Important:** You must have `Microsoft.Authorization/locks/delete` permission to remove locks (typically Owner or User Access Administrator role).

### Lock Management Best Practices

1. **Document locks:** Always add notes explaining why lock exists
2. **Use descriptive names:** E.g., "Production-DB-Protection" not "Lock1"
3. **Apply at appropriate scope:** Subscription lock protects everything but reduces flexibility
4. **Regular review:** Audit locks quarterly to ensure they're still needed
5. **Combine with Policy:** Use policy to require locks on critical resources

---

## Resource Tags

Tags are name-value pairs that help organize, manage, and query Azure resources.

### Tag Use Cases

| Use Case | Example Tags |
|----------|--------------|
| **Cost Tracking** | `CostCenter: IT-Operations`, `Project: Website-Redesign` |
| **Environment** | `Environment: Production`, `Environment: Development` |
| **Ownership** | `Owner: jane@contoso.com`, `Team: Platform-Engineering` |
| **Automation** | `AutoShutdown: Enabled`, `BackupSchedule: Daily` |
| **Compliance** | `DataClassification: Confidential`, `Compliance: HIPAA` |

### Tag Limitations

- **Maximum tags per resource:** 50
- **Tag name max length:** 512 characters (128 for storage accounts)
- **Tag value max length:** 256 characters
- **Case sensitivity:** Tag names are case-insensitive, values are case-sensitive
- **Unsupported resources:** Classic resources (Cloud Services, classic VMs)
- **Inheritance:** Tags do NOT automatically inherit from parent scopes

**Exam Alert:** Tags don't inherit from resource groups to resources by default, but Cost Management can enable tag inheritance for cost reporting purposes.

### Applying Tags via Portal

1. Navigate to the resource or resource group
2. Select **Tags** in the left menu
3. Enter **Name** and **Value**
4. Click **Save**

**Bulk Tagging:**
1. Navigate to **All resources**
2. Select multiple resources (checkboxes)
3. Click **Assign tags**
4. Apply tags to all selected resources

### Tag Policies

**Common Policy: Require Tag on Resources**

Enforces that resources must have specific tags:

```json
{
  "if": {
    "field": "tags['Environment']",
    "exists": "false"
  },
  "then": {
    "effect": "deny"
  }
}
```

**Common Policy: Inherit Tag from Resource Group**

Uses Modify effect to copy resource group tags to resources:

```json
{
  "if": {
    "allOf": [
      {
        "field": "tags['Environment']",
        "exists": "false"
      },
      {
        "value": "[resourceGroup().tags['Environment']]",
        "notEquals": ""
      }
    ]
  },
  "then": {
    "effect": "modify",
    "details": {
      "roleDefinitionIds": [
        "/providers/Microsoft.Authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c"
      ],
      "operations": [
        {
          "operation": "add",
          "field": "tags['Environment']",
          "value": "[resourceGroup().tags['Environment']]"
        }
      ]
    }
  }
}
```

### Tag Inheritance in Cost Management

While tags don't inherit automatically for resource management, Cost Management can enable tag inheritance for cost reporting:

1. Navigate to **Cost Management + Billing**
2. Select **Cost Management** → **Settings**
3. Enable **Tag inheritance**
4. Select which resource group and subscription tags to inherit
5. Wait 24 hours for cost data to reflect inherited tags

**Use Case:** Finance tags billing chargeback by CostCenter tag on resource groups. With tag inheritance, all child resource costs show the CostCenter tag even if resources don't have the tag applied directly.

### Querying Resources by Tags

**Azure Resource Graph Query:**
```kusto
Resources
| where tags['Environment'] == 'Production'
| project name, type, location, resourceGroup
```

**CLI:**
```bash
az resource list --tag Environment=Production
```

---

## Management Groups

Management groups provide a governance scope above subscriptions, allowing policy and access management at scale.

### Management Group Hierarchy

```
Root Management Group (Tenant)
├── Production Management Group
│   ├── Prod-Subscription-1
│   └── Prod-Subscription-2
├── Development Management Group
│   ├── Dev-Subscription-1
│   └── Dev-Subscription-2
└── Sandbox Management Group
    └── Sandbox-Subscription
```

### Key Characteristics

- **Maximum depth:** 6 levels (not including root or subscription level)
- **Root management group:** Automatically created, named after tenant ID
- **Parent assignment:** Each subscription/management group can have only one parent
- **Policy inheritance:** Policies assigned to management group apply to all child subscriptions
- **RBAC inheritance:** Role assignments inherit down the hierarchy
- **Moving subscriptions:** Can move between management groups (requires appropriate permissions)

**Exam Scenario:** If you assign a policy at the root management group, it applies to ALL subscriptions in the tenant. Use exclusions for exceptions.

### Creating Management Groups

**Portal Steps:**
1. Navigate to **Management groups**
2. Click **+ Add management group**
3. Configure:
   - **Management group ID:** Unique identifier (cannot be changed)
   - **Management group display name:** Friendly name (can be changed)
   - **Parent management group:** Select parent (default: root)
4. Click **Submit**

### Moving Subscriptions

**Portal Steps:**
1. Navigate to **Management groups**
2. Select the target management group
3. Click **Subscription** → **+ Add**
4. Select the subscription to move
5. Click **Save**

**Prerequisites:**
- Management Group Contributor role on target management group
- Owner role on subscription being moved

### Management Group Best Practices

1. **Logical hierarchy:** Organize by business unit, geography, or environment
2. **Shallow hierarchy:** Keep under 4 levels for manageability
3. **Policy placement:** Apply policies at the highest appropriate level
4. **Naming convention:** Use consistent IDs (e.g., mg-prod-eastus, mg-dev-westus)
5. **RBAC strategy:** Assign roles at management group level for broad access, subscription level for specific access

**Example Hierarchy:**
```
Root
├── mg-production (policies: encryption required, allowed regions)
│   ├── mg-prod-apps (policies: mandatory tags)
│   └── mg-prod-data (policies: SQL auditing required)
└── mg-non-production (policies: auto-shutdown enabled)
    ├── mg-dev
    └── mg-test
```

---

## Subscription Management

Azure subscriptions provide authenticated and authorized access to Azure resources, with billing and cost tracking.

### Subscription Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Free** | $200 credit for 30 days | Trial and learning |
| **Pay-As-You-Go** | Monthly billing based on usage | Production workloads |
| **Enterprise Agreement (EA)** | Annual commitment with discounts | Large organizations |
| **Microsoft Customer Agreement (MCA)** | Modern purchasing agreement | Flexible enterprise billing |
| **CSP (Cloud Solution Provider)** | Managed by partner | Managed services |
| **Dev/Test** | Discounted rates for non-production | Development and testing |

### Subscription States

| State | Description | Access |
|-------|-------------|--------|
| **Active** | Normal operational state | Full access |
| **Warned** | Payment overdue | Full access |
| **Disabled** | Payment failed or credit exhausted | Read-only access |
| **Deleted** | Subscription deleted, 90-day recovery period | No access |

### Subscription Limits

Azure has default limits (quotas) that can be increased by support request:

| Resource | Default Limit | Maximum Limit |
|----------|---------------|---------------|
| **Resource groups per subscription** | 980 | 980 |
| **Resources per resource group** | 800 | 800 |
| **Virtual Networks** | 1,000 | 1,000 |
| **VMs per subscription** | 25,000 | 25,000 |
| **Storage accounts per subscription** | 250 | 500 |

**Exam Tip:** If you hit subscription limits, create additional subscriptions or request quota increases via support ticket.

### Transferring Subscriptions

**Scenarios:**
- Changing billing ownership (e.g., moving from personal to company EA)
- Transferring to different Microsoft Entra tenant
- Moving between partners (CSP)

**Process:**
1. Navigate to **Cost Management + Billing**
2. Select the subscription
3. Click **Transfer billing ownership**
4. Enter recipient's email
5. Recipient accepts transfer

**Important:** Transferring to different tenant requires updating RBAC and resource configurations.

### Subscription Billing Structure (MCA)

```
Billing Account
└── Billing Profiles (payment method, invoice)
    └── Invoice Sections (cost grouping)
        └── Subscriptions
```

**Use Case:** Large enterprise has one billing account, separate billing profiles for each division, invoice sections for each department, multiple subscriptions per department.

---

## Cost Management

Azure Cost Management provides tools to monitor, allocate, and optimize cloud spending.

### Cost Analysis

View and analyze costs using various pivots:

**Common Views:**
- **Accumulated costs:** Total spending over time
- **Daily costs:** Day-by-day cost breakdown
- **Cost by service:** Which Azure services cost most
- **Cost by resource:** Individual resource costs
- **Cost by resource group:** Costs grouped by RG
- **Cost by tag:** Costs categorized by tags

**Portal Steps:**
1. Navigate to **Cost Management + Billing**
2. Select **Cost Management** → **Cost analysis**
3. Select **Scope** (billing account, subscription, resource group)
4. Choose **View** (accumulated, daily, forecast)
5. **Group by:** Service, Resource, Tag, Location
6. Apply **Filters:** Date range, tags, resource groups
7. **Download** or **Save view** for reuse

**Exam Tip:** Cost data has up to 24-hour delay. Forecasts predict spending based on usage trends.

### Budgets

Budgets proactively monitor spending and send alerts when thresholds are exceeded.

**Creating a Budget:**
1. Navigate to **Cost Management + Billing**
2. Select **Cost Management** → **Budgets**
3. Click **+ Add**
4. Configure:
   - **Scope:** Subscription or resource group
   - **Budget name:** Descriptive name
   - **Reset period:** Monthly, Quarterly, or Annually
   - **Creation date:** When budget starts
   - **Expiration date:** When budget ends (optional)
   - **Amount:** Budget limit (e.g., $1000/month)
5. Configure **Alert conditions:**
   - **Type:** Actual (spent) or Forecasted (predicted)
   - **% of budget:** 50%, 80%, 90%, 100%
   - **Action group:** Optional automation (email, webhook, Logic App)
   - **Alert recipients:** Email addresses
6. Click **Create**

**Alert Types:**
- **Actual:** Alert when actual spending crosses threshold
- **Forecasted:** Alert when projected spending will cross threshold by end of period

**Exam Scenario:** Budget alerts are informational only—they do NOT stop resource provisioning. Use Azure Policy to enforce spending controls.

### Cost Allocation

**Tagging Strategy:**
- Apply consistent tags: CostCenter, Project, Environment, Owner
- Enable tag inheritance in Cost Management
- Use Azure Policy to enforce required tags

**Allocation Rules:**
Cost Management can split shared costs (e.g., network, management) proportionally:

1. Navigate to **Cost Management** → **Cost allocation rules**
2. Click **+ Create**
3. Define:
   - **Source:** Shared resource or service
   - **Allocation method:** Proportional or even split
   - **Allocation basis:** Tag, resource group, or subscription
4. Click **Create**

**Use Case:** Company has shared networking infrastructure ($10K/month). Cost allocation splits this proportionally across business units based on their resource count.

### Exports

Automatically export cost data to storage accounts for custom reporting:

1. Navigate to **Cost Management** → **Exports**
2. Click **+ Add**
3. Configure:
   - **Export name:** Descriptive name
   - **Export type:** Actual cost or Amortized cost
   - **Frequency:** Daily, Weekly, Monthly
   - **Scope:** Subscription, resource group
   - **Storage account:** Destination for CSV files
   - **Container:** Blob container name
   - **Directory:** Optional folder path
4. Click **Create**

**Export Types:**
- **Actual cost:** Shows costs when incurred
- **Amortized cost:** Spreads reservation/savings plan costs over usage period

### Cost Optimization Recommendations

Azure Advisor provides cost reduction recommendations:

**Common Recommendations:**
- Right-size or shutdown underutilized VMs
- Delete unattached disks
- Use reserved instances for predictable workloads
- Configure auto-shutdown for dev/test VMs
- Move Blob storage to cooler tiers

**Viewing Recommendations:**
1. Navigate to **Advisor**
2. Select **Cost** tab
3. Review recommendations
4. Click recommendation for details and remediation steps

**Potential Savings:** Can reduce costs by 20-40% for most organizations.

### Reservation and Savings Plans

**Azure Reservations:**
- 1-year or 3-year commitment for specific services (VMs, SQL, Cosmos DB)
- Discounts up to 72% vs. pay-as-you-go
- Requires upfront or monthly payment
- Applies automatically to matching resources

**Azure Savings Plans:**
- Flexible hourly commitment for compute services
- Discounts up to 65%
- Not tied to specific VM sizes or regions
- More flexibility than reservations

**Exam Tip:** Reservations save more but are less flexible. Savings plans are good for variable workloads.

---

## Governance Best Practices

### Naming Conventions

**Recommended Format:** `<resource-type>-<workload/app>-<environment>-<region>-<instance>`

**Examples:**
- `vm-webapp-prod-eastus-001`
- `st-logs-shared-westus-01` (storage accounts have restrictions)
- `rg-networking-prod-eastus`

**Resource Abbreviations:**
| Resource | Abbreviation |
|----------|--------------|
| Resource Group | rg |
| Virtual Machine | vm |
| Virtual Network | vnet |
| Storage Account | st |
| Key Vault | kv |
| App Service | app |
| SQL Database | sqldb |

### Resource Organization

**Multi-Subscription Strategy:**

```
Tenant
├── Management Group: Production
│   ├── Subscription: Prod-Apps
│   ├── Subscription: Prod-Data
│   └── Subscription: Prod-Network
└── Management Group: Non-Production
    ├── Subscription: Dev
    ├── Subscription: Test
    └── Subscription: Sandbox
```

**Resource Group Strategy:**
- **By lifecycle:** Resources that are created/deleted together
- **By application:** All resources for one application
- **By environment:** Prod vs. Dev resources (alternative: use subscriptions)
- **By resource type:** All VMs in one RG (NOT recommended)

**Best Practice:** Organize resource groups by application and lifecycle. Example: `rg-webapp-prod` contains all resources for the production web application.

### Tagging Strategy

**Mandatory Tags (Enforced by Policy):**
```
Environment: Production | Development | Test
CostCenter: IT-Ops | Marketing | Finance
Owner: email@contoso.com
Application: WebApp | API | Database
```

**Optional Tags:**
```
Criticality: High | Medium | Low
DataClassification: Public | Internal | Confidential
BackupRequired: Yes | No
AutoShutdown: Enabled | Disabled
```

**Implementation:**
1. Create custom policy requiring mandatory tags
2. Assign policy at subscription or management group level
3. Use Modify policy effect to inherit tags from resource groups
4. Enable tag inheritance in Cost Management for billing reports

### Governance Enforcement Levels

**Level 1: Management Group**
- Broad organizational policies (e.g., allowed regions, required encryption)
- RBAC for global administrators
- Applies to all subscriptions

**Level 2: Subscription**
- Environment-specific policies (e.g., production requires backups)
- RBAC for subscription administrators
- Cost budgets and alerts

**Level 3: Resource Group**
- Application-specific policies (e.g., mandatory tags)
- Resource locks on critical resources
- RBAC for application teams

**Level 4: Resource**
- Individual resource locks
- Resource-specific RBAC
- Fine-grained access control

---

## PowerShell & CLI Commands

### Azure Policy Commands

#### PowerShell

```powershell
# List all policy definitions
Get-AzPolicyDefinition

# Get specific policy definition
Get-AzPolicyDefinition -Name 'allowed-locations'

# Create custom policy definition
$policyRule = @{
  if = @{
    field = 'location'
    notIn = @('eastus', 'westus')
  }
  then = @{
    effect = 'deny'
  }
}

New-AzPolicyDefinition `
  -Name 'allowed-locations-custom' `
  -DisplayName 'Allowed Locations (Custom)' `
  -Policy ($policyRule | ConvertTo-Json -Depth 10) `
  -Description 'Restricts resource deployment to specific regions'

# Assign policy to subscription
$subscription = Get-AzSubscription -SubscriptionName 'Production'
$policy = Get-AzPolicyDefinition -Name 'allowed-locations'

New-AzPolicyAssignment `
  -Name 'restrict-locations' `
  -DisplayName 'Restrict Resource Locations' `
  -Scope "/subscriptions/$($subscription.Id)" `
  -PolicyDefinition $policy `
  -PolicyParameterObject @{listOfAllowedLocations = @('eastus', 'westus')}

# Assign policy to resource group
$rg = Get-AzResourceGroup -Name 'rg-production'
New-AzPolicyAssignment `
  -Name 'require-tags' `
  -Scope $rg.ResourceId `
  -PolicyDefinition (Get-AzPolicyDefinition -Name 'require-tag')

# Create initiative (policy set)
$policies = @(
  @{policyDefinitionId = '/providers/Microsoft.Authorization/policyDefinitions/policy1-id'},
  @{policyDefinitionId = '/providers/Microsoft.Authorization/policyDefinitions/policy2-id'}
)

New-AzPolicySetDefinition `
  -Name 'security-baseline' `
  -DisplayName 'Security Baseline Initiative' `
  -PolicyDefinition ($policies | ConvertTo-Json -Depth 10)

# Check policy compliance
Get-AzPolicyState -ResourceGroupName 'rg-production'

# Get non-compliant resources
Get-AzPolicyState | Where-Object { $_.ComplianceState -eq 'NonCompliant' }

# Create remediation task
Start-AzPolicyRemediation `
  -Name 'remediate-tags' `
  -PolicyAssignmentId '/subscriptions/.../assignments/require-tags' `
  -ResourceGroupName 'rg-production'

# Remove policy assignment
Remove-AzPolicyAssignment -Name 'restrict-locations' -Scope "/subscriptions/$($subscription.Id)"
```

#### Azure CLI

```bash
# List all policy definitions
az policy definition list --output table

# Get specific policy definition
az policy definition show --name 'allowed-locations'

# Create custom policy definition
az policy definition create \
  --name 'allowed-locations-custom' \
  --display-name 'Allowed Locations (Custom)' \
  --description 'Restricts resource deployment to specific regions' \
  --rules '{
    "if": {
      "field": "location",
      "notIn": ["eastus", "westus"]
    },
    "then": {
      "effect": "deny"
    }
  }' \
  --mode Indexed

# Assign policy to subscription
az policy assignment create \
  --name 'restrict-locations' \
  --display-name 'Restrict Resource Locations' \
  --scope "/subscriptions/<subscription-id>" \
  --policy 'allowed-locations' \
  --params '{
    "listOfAllowedLocations": {
      "value": ["eastus", "westus"]
    }
  }'

# Assign policy to resource group
az policy assignment create \
  --name 'require-tags' \
  --scope "/subscriptions/<subscription-id>/resourceGroups/rg-production" \
  --policy 'require-tag'

# Create initiative (policy set)
az policy set-definition create \
  --name 'security-baseline' \
  --display-name 'Security Baseline Initiative' \
  --definitions '[
    {"policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/policy1-id"},
    {"policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/policy2-id"}
  ]'

# Check policy compliance
az policy state list --resource-group 'rg-production'

# Get non-compliant resources
az policy state list --filter "ComplianceState eq 'NonCompliant'" --output table

# Create remediation task
az policy remediation create \
  --name 'remediate-tags' \
  --policy-assignment '/subscriptions/.../assignments/require-tags' \
  --resource-group 'rg-production'

# Remove policy assignment
az policy assignment delete --name 'restrict-locations'
```

### Resource Lock Commands

#### PowerShell

```powershell
# Create delete lock on resource group
$rg = Get-AzResourceGroup -Name 'rg-production'
New-AzResourceLock `
  -LockName 'production-lock' `
  -LockLevel CanNotDelete `
  -ResourceGroupName $rg.ResourceGroupName `
  -LockNotes 'Prevent accidental deletion of production resources'

# Create read-only lock on resource
New-AzResourceLock `
  -LockName 'database-readonly' `
  -LockLevel ReadOnly `
  -ResourceGroupName 'rg-production' `
  -ResourceName 'sqlserver-prod' `
  -ResourceType 'Microsoft.Sql/servers'

# Create lock on subscription
New-AzResourceLock `
  -LockName 'subscription-lock' `
  -LockLevel CanNotDelete `
  -Scope "/subscriptions/<subscription-id>" `
  -LockNotes 'Prevent subscription deletion'

# List all locks in resource group
Get-AzResourceLock -ResourceGroupName 'rg-production'

# List all locks in subscription
Get-AzResourceLock

# Remove lock
Remove-AzResourceLock `
  -LockName 'production-lock' `
  -ResourceGroupName 'rg-production' `
  -Force
```

#### Azure CLI

```bash
# Create delete lock on resource group
az lock create \
  --name 'production-lock' \
  --lock-type CanNotDelete \
  --resource-group 'rg-production' \
  --notes 'Prevent accidental deletion of production resources'

# Create read-only lock on resource
az lock create \
  --name 'database-readonly' \
  --lock-type ReadOnly \
  --resource-group 'rg-production' \
  --resource-name 'sqlserver-prod' \
  --resource-type 'Microsoft.Sql/servers'

# Create lock on subscription
az lock create \
  --name 'subscription-lock' \
  --lock-type CanNotDelete \
  --subscription '<subscription-id>' \
  --notes 'Prevent subscription deletion'

# List all locks in resource group
az lock list --resource-group 'rg-production' --output table

# List all locks in subscription
az lock list --output table

# Remove lock
az lock delete \
  --name 'production-lock' \
  --resource-group 'rg-production'
```

### Resource Tag Commands

#### PowerShell

```powershell
# Apply tags to resource group
$tags = @{
  Environment = 'Production'
  CostCenter = 'IT-Operations'
  Owner = 'admin@contoso.com'
}
Set-AzResourceGroup -Name 'rg-production' -Tag $tags

# Add tags to existing tags (merge)
$rg = Get-AzResourceGroup -Name 'rg-production'
$newTags = @{Application = 'WebApp'}
$mergedTags = $rg.Tags + $newTags
Set-AzResourceGroup -Name 'rg-production' -Tag $mergedTags

# Apply tags to resource
$vm = Get-AzVM -ResourceGroupName 'rg-production' -Name 'vm-web-01'
$tags = @{
  Environment = 'Production'
  BackupRequired = 'Yes'
}
Update-AzTag -ResourceId $vm.Id -Tag $tags -Operation Merge

# Get resources by tag
Get-AzResource -TagName 'Environment' -TagValue 'Production'

# Remove specific tag
$rg = Get-AzResourceGroup -Name 'rg-production'
$rg.Tags.Remove('OldTag')
Set-AzResourceGroup -Name 'rg-production' -Tag $rg.Tags

# Remove all tags
Set-AzResourceGroup -Name 'rg-production' -Tag @{}

# Copy tags from resource group to resources (bulk operation)
$rg = Get-AzResourceGroup -Name 'rg-production'
$resources = Get-AzResource -ResourceGroupName $rg.ResourceGroupName
foreach ($resource in $resources) {
  Update-AzTag -ResourceId $resource.ResourceId -Tag $rg.Tags -Operation Merge
}

# List all tag names and values in subscription
Get-AzTag
```

#### Azure CLI

```bash
# Apply tags to resource group
az group update \
  --name 'rg-production' \
  --tags Environment=Production CostCenter=IT-Operations Owner=admin@contoso.com

# Add tags to existing tags (merge)
az group update \
  --name 'rg-production' \
  --set tags.Application=WebApp

# Apply tags to resource
az vm update \
  --resource-group 'rg-production' \
  --name 'vm-web-01' \
  --set tags.Environment=Production tags.BackupRequired=Yes

# Get resources by tag
az resource list --tag Environment=Production --output table

# Remove specific tag
az group update \
  --name 'rg-production' \
  --remove tags.OldTag

# Remove all tags
az group update \
  --name 'rg-production' \
  --set tags={}

# List all tag names in subscription
az tag list --output table
```

### Management Group Commands

#### PowerShell

```powershell
# Create management group
New-AzManagementGroup -GroupId 'mg-production' -DisplayName 'Production Management Group'

# Create nested management group
New-AzManagementGroup `
  -GroupId 'mg-prod-apps' `
  -DisplayName 'Production Applications' `
  -ParentId 'mg-production'

# List all management groups
Get-AzManagementGroup

# Get management group details
Get-AzManagementGroup -GroupId 'mg-production' -Expand

# Move subscription to management group
New-AzManagementGroupSubscription `
  -GroupId 'mg-production' `
  -SubscriptionId '<subscription-id>'

# Remove subscription from management group (moves to root)
Remove-AzManagementGroupSubscription `
  -GroupId 'mg-production' `
  -SubscriptionId '<subscription-id>'

# Update management group
Update-AzManagementGroup -GroupId 'mg-production' -DisplayName 'Production (Updated)'

# Remove management group (must be empty)
Remove-AzManagementGroup -GroupId 'mg-production'
```

#### Azure CLI

```bash
# Create management group
az account management-group create \
  --name 'mg-production' \
  --display-name 'Production Management Group'

# Create nested management group
az account management-group create \
  --name 'mg-prod-apps' \
  --display-name 'Production Applications' \
  --parent 'mg-production'

# List all management groups
az account management-group list --output table

# Get management group details
az account management-group show --name 'mg-production'

# Move subscription to management group
az account management-group subscription add \
  --name 'mg-production' \
  --subscription '<subscription-id>'

# Remove subscription from management group
az account management-group subscription remove \
  --name 'mg-production' \
  --subscription '<subscription-id>'

# Update management group
az account management-group update \
  --name 'mg-production' \
  --display-name 'Production (Updated)'

# Remove management group (must be empty)
az account management-group delete --name 'mg-production'
```

### Cost Management Commands

#### PowerShell

```powershell
# Get cost for current month
$startDate = (Get-Date -Day 1).ToString('yyyy-MM-dd')
$endDate = (Get-Date).ToString('yyyy-MM-dd')

Get-AzConsumptionUsageDetail `
  -StartDate $startDate `
  -EndDate $endDate

# Create budget
$subscription = Get-AzSubscription -SubscriptionName 'Production'
$emailContacts = @('admin@contoso.com', 'finance@contoso.com')

New-AzConsumptionBudget `
  -Name 'monthly-budget' `
  -Amount 10000 `
  -Category Cost `
  -StartDate '2024-01-01' `
  -EndDate '2025-12-31' `
  -TimeGrain Monthly `
  -ContactEmail $emailContacts `
  -NotificationKey 'Threshold80' `
  -NotificationThreshold 80 `
  -NotificationEnabled

# List all budgets
Get-AzConsumptionBudget

# Update budget
$budget = Get-AzConsumptionBudget -Name 'monthly-budget'
$budget.Amount = 12000
$budget | Set-AzConsumptionBudget

# Remove budget
Remove-AzConsumptionBudget -Name 'monthly-budget'
```

#### Azure CLI

```bash
# Get cost for current month
az consumption usage list \
  --start-date 2024-01-01 \
  --end-date 2024-01-31 \
  --output table

# Create budget
az consumption budget create \
  --budget-name 'monthly-budget' \
  --amount 10000 \
  --category cost \
  --time-grain monthly \
  --start-date 2024-01-01 \
  --end-date 2025-12-31 \
  --notifications '{
    "Actual_GreaterThan_80_Percent": {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 80,
      "contactEmails": ["admin@contoso.com", "finance@contoso.com"]
    }
  }'

# List all budgets
az consumption budget list --output table

# Update budget
az consumption budget update \
  --budget-name 'monthly-budget' \
  --amount 12000

# Delete budget
az consumption budget delete --budget-name 'monthly-budget'

# Query costs with Azure Resource Graph
az graph query -q "
  CostManagementResources
  | where type == 'microsoft.costmanagement/query'
  | summarize TotalCost = sum(todouble(properties.cost))
"
```

---

## Exam Tips

### Key Concepts for AZ-104

1. **Policy vs. RBAC:**
   - **Policy:** Controls what resources can be created and how they're configured
   - **RBAC:** Controls who can create and manage resources
   - **Exam Scenario:** User has Owner role but can't deploy VM to disallowed region (Policy denies)

2. **Policy Effects Priority:**
   - If multiple policies apply: **Deny** > **Audit** > **Append**
   - **Disabled** policies are not evaluated
   - **Exam Question:** Resource has conflicting policies—which takes precedence?

3. **Lock Inheritance:**
   - Locks inherit from parent to child scopes
   - **ReadOnly** lock on resource group prevents modifying ANY resource in the group
   - **Exam Scenario:** User can't start VM—check for ReadOnly lock on RG

4. **Tag Inheritance:**
   - Tags do NOT inherit automatically
   - Use **Modify** policy effect or Cost Management tag inheritance
   - **Exam Trap:** Applying tag to RG doesn't automatically tag resources

5. **Management Group Limits:**
   - Maximum 6 levels deep (not counting root and subscription)
   - Each subscription can have only one parent
   - **Exam Calculation:** Root → Level 1 → Level 2 → Level 3 → Level 4 → Level 5 → Subscription

6. **Policy Remediation:**
   - **DeployIfNotExists** and **Modify** effects require managed identity
   - Remediation tasks apply to existing resources
   - **Exam Scenario:** Policy assigned but existing resources still non-compliant—run remediation task

7. **Cost Management Timing:**
   - Cost data has 8-24 hour delay
   - Budgets send alerts but don't prevent spending
   - **Exam Tip:** Use Policy to enforce spending controls (e.g., deny expensive VM SKUs)

8. **Subscription Limits:**
   - Each subscription has quotas (e.g., 25,000 VMs)
   - Request quota increases via support ticket
   - **Exam Scenario:** Can't deploy more VMs—check quota limits

9. **Scope Hierarchy:**
   ```
   Management Group (broadest)
   └── Subscription
       └── Resource Group
           └── Resource (narrowest)
   ```
   - Policy/RBAC assigned at higher scope applies to all children
   - More specific assignments override less specific

10. **Policy Assignment Components:**
    - **Definition:** The rule
    - **Assignment:** Applies definition to scope
    - **Parameters:** Values for the rule (e.g., allowed locations)
    - **Exemption:** Exceptions to the assignment

### Common Exam Scenarios

**Scenario 1: Prevent Resource Deletion**
- **Question:** Production resources accidentally deleted. How to prevent?
- **Answer:** Apply **CanNotDelete** lock at resource group level
- **Why:** Locks override RBAC permissions, even for Owners

**Scenario 2: Track Costs by Department**
- **Question:** Finance needs cost breakdown by department
- **Answer:** 
  1. Create `Department` tag with policy enforcement
  2. Enable tag inheritance in Cost Management
  3. Use Cost Analysis grouped by Department tag
- **Why:** Tags are the only way to add custom metadata for cost tracking

**Scenario 3: Restrict VM Deployment Regions**
- **Question:** Company policy: Deploy only to East US and West US
- **Answer:** Assign "Allowed locations" policy with Deny effect at subscription level
- **Why:** Policy prevents non-compliant deployments, Deny effect blocks attempts

**Scenario 4: Existing Resources Don't Have Required Tag**
- **Question:** Policy requires `CostCenter` tag but existing VMs don't have it
- **Answer:** 
  1. Verify policy effect is Modify or use separate Modify policy
  2. Create remediation task for the policy assignment
- **Why:** New resources will have tag automatically; existing need remediation

**Scenario 5: User Can't Delete Resource Despite Owner Role**
- **Question:** User has Owner role but can't delete VM, gets "Locked" error
- **Answer:** Check for **CanNotDelete** or **ReadOnly** lock on resource, RG, or subscription
- **Why:** Locks override RBAC roles

**Scenario 6: Organize 100 Subscriptions**
- **Question:** Large organization has 100 subscriptions, needs governance structure
- **Answer:** 
  1. Create management group hierarchy (e.g., by business unit)
  2. Move subscriptions to appropriate management groups
  3. Assign policies at management group level
- **Why:** Management groups provide scale and inheritance

**Scenario 7: Auto-Enable Diagnostic Settings**
- **Question:** Require all VMs to send logs to Log Analytics
- **Answer:** 
  1. Assign "Deploy Diagnostic Settings for VMs" policy (DeployIfNotExists effect)
  2. Configure managed identity with appropriate RBAC
  3. Run remediation task for existing VMs
- **Why:** DeployIfNotExists automatically creates diagnostic settings

**Scenario 8: Budget Alert But Spending Continues**
- **Question:** Budget exceeded 100% but new resources still deploying
- **Answer:** Budgets only alert—they don't prevent spending. Use Azure Policy to deny expensive resources or implement automation via Action Groups
- **Why:** Budgets are monitoring tools, not enforcement tools

---

## Common Scenarios & Troubleshooting

### Scenario: Policy Not Being Enforced

**Symptoms:**
- Resources deployed in violation of policy
- Policy shows as assigned but not evaluated

**Troubleshooting Steps:**
1. **Verify scope:** Check policy assignment scope includes the resource
2. **Check exclusions:** Resource might be explicitly excluded
3. **Verify policy effect:** Disabled policies don't evaluate
4. **Wait for evaluation:** New assignments take 15-30 minutes
5. **Check exemptions:** Resource may have policy exemption
6. **Validate JSON:** Custom policy might have syntax errors

**Resolution:**
```powershell
# Force policy evaluation
Start-AzPolicyComplianceScan -ResourceGroupName 'rg-production'

# Check policy state for specific resource
Get-AzPolicyState -ResourceId '/subscriptions/.../resourceGroups/rg-production/providers/Microsoft.Compute/virtualMachines/vm-01'
```

### Scenario: Can't Remove Lock

**Symptoms:**
- "Forbidden" error when trying to delete lock
- Lock delete button grayed out

**Troubleshooting Steps:**
1. **Check permissions:** Requires `Microsoft.Authorization/locks/delete` permission
2. **Verify scope:** Lock might be inherited from parent (check RG/subscription)
3. **Check role:** Need Owner or User Access Administrator role
4. **Classic resources:** Classic locks can only be removed via classic portal

**Resolution:**
```powershell
# Find lock on resource
$resource = Get-AzResource -Name 'vm-01'
Get-AzResourceLock -ResourceId $resource.ResourceId

# Check parent locks
Get-AzResourceLock -ResourceGroupName 'rg-production'
Get-AzResourceLock -Scope "/subscriptions/<subscription-id>"

# Remove with proper scope
Remove-AzResourceLock -LockId '<lock-id>' -Force
```

### Scenario: Tags Not Appearing in Cost Analysis

**Symptoms:**
- Resources have tags but Cost Analysis doesn't show them
- Tag filter returns no results

**Troubleshooting Steps:**
1. **Wait for sync:** Tags can take 24 hours to appear in cost data
2. **Check tag support:** Some resources don't support tags (classic resources)
3. **Verify tag name:** Tag names are case-insensitive but check spelling
4. **Usage vs. resources:** Cost Analysis shows usage records, not all resources

**Resolution:**
- Enable tag inheritance in Cost Management settings
- Reapply tags if recently changed
- Download usage details CSV to verify tags

### Scenario: Policy Remediation Task Fails

**Symptoms:**
- Remediation task shows "Failed" status
- Some resources remediated, others failed

**Troubleshooting Steps:**
1. **Check managed identity:** Verify identity has required RBAC roles
2. **Review error messages:** Activity log shows specific failures
3. **Resource locks:** ReadOnly locks prevent remediation
4. **Resource state:** Can't modify deleted or deallocated resources
5. **Rate limiting:** Too many concurrent operations

**Resolution:**
```powershell
# Check remediation task status
Get-AzPolicyRemediation -Name 'remediation-01'

# Review detailed errors
$remediation = Get-AzPolicyRemediation -Name 'remediation-01'
$remediation.ProvisioningState
$remediation.StatusMessage

# Grant required role to managed identity
$policyAssignment = Get-AzPolicyAssignment -Name 'deploy-diagnostics'
New-AzRoleAssignment `
  -ObjectId $policyAssignment.Identity.PrincipalId `
  -RoleDefinitionName 'Contributor' `
  -Scope "/subscriptions/<subscription-id>"

# Retry remediation
Start-AzPolicyRemediation `
  -Name 'remediation-retry' `
  -PolicyAssignmentId $policyAssignment.PolicyAssignmentId
```

### Scenario: Budget Alerts Not Received

**Symptoms:**
- Budget threshold exceeded but no email received
- Action group not triggered

**Troubleshooting Steps:**
1. **Check email addresses:** Verify correct emails in budget configuration
2. **Spam folder:** Budget emails might be filtered
3. **Action group status:** Verify action group is enabled
4. **Alert conditions:** Check if using Actual vs. Forecasted correctly
5. **Rate limiting:** Azure limits alert frequency (one per day per threshold)

**Resolution:**
```powershell
# Verify budget configuration
Get-AzConsumptionBudget -Name 'monthly-budget'

# Check action group
Get-AzActionGroup -ResourceGroupName 'rg-monitoring' -Name 'budget-alerts'

# Test action group
Test-AzActionGroup -ResourceGroupName 'rg-monitoring' -ActionGroupName 'budget-alerts'

# Recreate budget with verified settings
Remove-AzConsumptionBudget -Name 'monthly-budget'
New-AzConsumptionBudget -Name 'monthly-budget' -Amount 10000 # ... full parameters
```

### Scenario: Management Group Subscription Move Fails

**Symptoms:**
- Error: "Subscription cannot be moved"
- Permission denied

**Troubleshooting Steps:**
1. **Check permissions:** Need Owner on subscription AND Management Group Contributor on target MG
2. **Subscription state:** Can't move disabled subscriptions
3. **EA restrictions:** Enterprise Agreement subscriptions may have limitations
4. **Transfer pending:** Can't move during billing transfer
5. **Inherited policies:** Target MG policies might block the subscription's resources

**Resolution:**
```powershell
# Verify permissions
Get-AzRoleAssignment -ObjectId <user-object-id> -Scope "/providers/Microsoft.Management/managementGroups/mg-target"

# Check subscription state
Get-AzSubscription -SubscriptionId <subscription-id>

# Move subscription
New-AzManagementGroupSubscription `
  -GroupId 'mg-target' `
  -SubscriptionId <subscription-id>

# If permission issues, grant required roles
New-AzRoleAssignment `
  -ObjectId <user-object-id> `
  -RoleDefinitionName 'Management Group Contributor' `
  -Scope "/providers/Microsoft.Management/managementGroups/mg-target"
```

---

## Related Topics

### Linked Study Materials

- **[Manage AAD Objects](./Manage%20AAD%20Objects.md):** User and group management, prerequisites for RBAC
- **[Manage RBAC](./Manage%20RBAC.md):** Access control, works alongside governance
- **[Monitor Resources with Azure Monitor](../Monitor%20and%20Backup%20Azure%20Resources/Monitor%20Resources%20with%20Azure%20Monitor.md):** Logging and alerting for governance
- **[Implement Backup and Recovery](../Monitor%20and%20Backup%20Azure%20Resources/Implement%20Backup%20and%20Recovery.md):** Protect resources governed by policies

### Azure Services Integration

- **Azure Blueprints:** Orchestrate deployment of governance artifacts (policies, roles, ARM templates)
- **Microsoft Defender for Cloud:** Security policies and compliance dashboards
- **Azure Advisor:** Cost optimization and governance recommendations
- **Azure Resource Graph:** Query resources across subscriptions for compliance reporting
- **Azure Automation:** Automate governance tasks (e.g., tag enforcement, resource cleanup)

### Advanced Topics (Beyond AZ-104)

- **Azure Landing Zones:** Enterprise-scale governance architecture
- **FinOps Practices:** Cloud cost optimization and financial management
- **Sovereign Cloud:** Specialized governance for government clouds
- **Policy as Code:** Managing policies with CI/CD pipelines
- **Custom RBAC Roles:** Creating granular access control beyond built-in roles

### Documentation Links

- [Azure Policy Documentation](https://learn.microsoft.com/en-us/azure/governance/policy/)
- [Management Groups Overview](https://learn.microsoft.com/en-us/azure/governance/management-groups/)
- [Azure Cost Management](https://learn.microsoft.com/en-us/azure/cost-management-billing/)
- [Resource Locks](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/lock-resources)
- [Resource Tags](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/tag-resources)
- [Cloud Adoption Framework - Governance](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/govern/)

---

**Study Checkpoint:** You should now be able to implement Azure Policy for compliance, use resource locks to prevent deletion, apply tags for organization, structure management groups, manage subscriptions, and control costs with budgets. Practice creating policies, applying locks, and analyzing costs in the Azure portal.

**Next Steps:** Review [Implement and Manage Storage](../Implement%20and%20Manage%20Storage/) for storage governance and [Configure VMs](../Deploy%20and%20Manage%20Azure%20Compute%20Resources/Configure%20VMs.md) to apply governance to compute resources.
