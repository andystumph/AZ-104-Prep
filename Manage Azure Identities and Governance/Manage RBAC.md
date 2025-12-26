# Manage Role-Based Access Control (RBAC)

> **Exam Weight:** Part of Manage Azure identities and governance (20-25%)

## Table of Contents
- [Overview](#overview)
- [RBAC Fundamentals](#rbac-fundamentals)
- [Built-in Azure Roles](#built-in-azure-roles)
- [Custom Roles](#custom-roles)
- [Role Assignments](#role-assignments)
- [Scope Hierarchy](#scope-hierarchy)
- [Interpreting Access Assignments](#interpreting-access-assignments)
- [PowerShell and CLI Commands](#powershell-and-cli-commands)
- [Best Practices](#best-practices)
- [Exam Tips](#exam-tips)

---

## Overview

**Azure Role-Based Access Control (RBAC)** is an authorization system built on Azure Resource Manager that provides fine-grained access management of Azure resources.

### Key Benefits
- **Segregation of duties** within your team
- **Grant only the access** users need to perform their jobs
- **Enable access** to Azure portal and control access to resources
- **Define what actions** users can perform at a specific scope

### RBAC vs Microsoft Entra ID Roles

| Azure RBAC | Microsoft Entra ID Roles |
|------------|-------------------------|
| Manages access to Azure resources | Manages access to Entra ID resources |
| Scope can be specified at multiple levels | Scope is at tenant level |
| Role information can be accessed via Portal, CLI, PowerShell, ARM templates | Role information via Azure AD Portal, Microsoft Graph |
| Examples: Owner, Contributor, Reader | Examples: Global Administrator, User Administrator |

---

## RBAC Fundamentals

### The Four Components of RBAC

1. **Security Principal**: Who is requesting access
   - User
   - Group
   - Service Principal (application)
   - Managed Identity

2. **Role Definition**: What actions can be performed
   - Collection of permissions
   - Built-in or custom roles

3. **Scope**: Where the access applies
   - Management group
   - Subscription
   - Resource group
   - Resource

4. **Role Assignment**: Combining principal + role + scope

```
[Security Principal] + [Role Definition] + [Scope] = Access
```

### How RBAC Works

When you access Azure resources, RBAC checks:
1. Does this principal have a role assignment at this scope?
2. What actions does the role allow (Actions)?
3. What actions does the role explicitly deny (NotActions)?
4. What data actions are allowed/denied (DataActions/NotDataActions)?

**Evaluation Logic:**
- Role assignments are **additive** (if you have multiple roles, you get all permissions)
- **Deny assignments** override allow assignments
- **NotActions** subtract from Actions

---

## Built-in Azure Roles

Azure provides over 100 built-in roles. Here are the most important ones for the AZ-104 exam:

### Fundamental Roles

#### Owner
- **Description**: Full access to all resources including the right to delegate access
- **Use Case**: Subscription administrators, resource group owners
- **Permissions**: `*` (all actions)

```json
{
  "Actions": ["*"],
  "NotActions": [],
  "DataActions": [],
  "NotDataActions": []
}
```

#### Contributor
- **Description**: Full access to all resources but cannot grant access to others
- **Use Case**: Developers, engineers who manage resources
- **Key Difference from Owner**: Cannot assign roles to others

```json
{
  "Actions": ["*"],
  "NotActions": [
    "Microsoft.Authorization/*/Delete",
    "Microsoft.Authorization/*/Write",
    "Microsoft.Authorization/elevateAccess/Action"
  ]
}
```

#### Reader
- **Description**: View all resources but cannot make changes
- **Use Case**: Auditors, monitoring users
- **Permissions**: `*/read`

```json
{
  "Actions": ["*/read"],
  "NotActions": []
}
```

#### User Access Administrator
- **Description**: Manage user access to Azure resources
- **Use Case**: Dedicated access managers
- **Permissions**: Manage role assignments only

### Resource-Specific Built-in Roles

#### Virtual Machine Contributor
- Create and manage VMs
- Cannot manage virtual network or storage account
- Cannot assign roles

#### Network Contributor
- Manage networks
- Cannot assign roles

#### Storage Account Contributor
- Manage storage accounts
- Cannot access storage account keys or data

#### Storage Blob Data Contributor
- Read, write, and delete Azure Storage containers and blobs
- This is a **data plane** role

#### Key Vault Contributor
- Manage key vaults
- Does NOT provide access to keys, secrets, or certificates (data plane)

#### Monitoring Contributor
- Read all monitoring data
- Update monitoring settings

### Viewing Built-in Roles

#### Azure Portal
1. Navigate to **Subscriptions** or **Resource Groups**
2. Select **Access control (IAM)**
3. Click **Roles** tab
4. View all available roles

#### PowerShell
```powershell
# List all role definitions
Get-AzRoleDefinition | Format-Table Name, Description

# Get specific role details
Get-AzRoleDefinition "Virtual Machine Contributor"

# List roles for a resource provider
Get-AzRoleDefinition | Where-Object {$_.Actions -like "*Microsoft.Compute*"}
```

#### Azure CLI
```bash
# List all roles
az role definition list --output table

# Get specific role
az role definition list --name "Contributor"

# Search for roles
az role definition list --custom-role-only false --output json | grep "Virtual Machine"
```

---

## Custom Roles

When built-in roles don't meet your needs, create custom roles.

### When to Use Custom Roles
- Need specific combination of permissions
- Want to restrict certain actions within a service
- Implement principle of least privilege
- Standardize access patterns in your organization

### Custom Role Structure

```json
{
  "Name": "Custom Role Name",
  "Id": "guid",
  "IsCustom": true,
  "Description": "Description of role",
  "Actions": [
    "Microsoft.Compute/*/read",
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action"
  ],
  "NotActions": [
    "Microsoft.Compute/virtualMachines/delete"
  ],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": [
    "/subscriptions/{subscriptionId}",
    "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}"
  ]
}
```

### Components Explained

- **Actions**: What management plane operations are allowed
- **NotActions**: What management plane operations are specifically denied
- **DataActions**: What data plane operations are allowed
- **NotDataActions**: What data plane operations are specifically denied
- **AssignableScopes**: Where this role can be assigned (subscription, resource group)

### Creating Custom Roles

#### Method 1: Clone and Modify Existing Role

```powershell
# Get existing role
$role = Get-AzRoleDefinition "Virtual Machine Contributor"

# Modify properties
$role.Id = $null
$role.Name = "Virtual Machine Operator"
$role.Description = "Can monitor and restart VMs"
$role.Actions.Clear()
$role.Actions.Add("Microsoft.Compute/*/read")
$role.Actions.Add("Microsoft.Compute/virtualMachines/start/action")
$role.Actions.Add("Microsoft.Compute/virtualMachines/restart/action")
$role.AssignableScopes.Clear()
$role.AssignableScopes.Add("/subscriptions/{subscription-id}")

# Create custom role
New-AzRoleDefinition -Role $role
```

#### Method 2: Create from JSON File

**customRole.json:**
```json
{
  "Name": "Virtual Machine Operator",
  "IsCustom": true,
  "Description": "Can monitor and restart virtual machines",
  "Actions": [
    "Microsoft.Compute/*/read",
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Storage/*/read",
    "Microsoft.Network/*/read",
    "Microsoft.Authorization/*/read",
    "Microsoft.Resources/subscriptions/resourceGroups/read",
    "Microsoft.Insights/alertRules/*",
    "Microsoft.Support/*"
  ],
  "NotActions": [],
  "AssignableScopes": [
    "/subscriptions/00000000-0000-0000-0000-000000000000"
  ]
}
```

```powershell
# Create from JSON
New-AzRoleDefinition -InputFile "customRole.json"
```

#### Azure CLI
```bash
# Create custom role from JSON
az role definition create --role-definition customRole.json

# Update existing custom role
az role definition update --role-definition customRole.json
```

### Common Custom Role Examples

#### 1. VM Start/Stop Operator
```json
{
  "Name": "VM Start/Stop Operator",
  "Actions": [
    "Microsoft.Compute/*/read",
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/deallocate/action"
  ],
  "NotActions": [],
  "AssignableScopes": ["/subscriptions/{sub-id}"]
}
```

#### 2. Storage Account Key Reader
```json
{
  "Name": "Storage Account Key Reader",
  "Actions": [
    "Microsoft.Storage/storageAccounts/listkeys/action",
    "Microsoft.Storage/storageAccounts/read"
  ],
  "NotActions": [],
  "AssignableScopes": ["/subscriptions/{sub-id}"]
}
```

#### 3. Network Security Group Manager
```json
{
  "Name": "NSG Manager",
  "Actions": [
    "Microsoft.Network/networkSecurityGroups/*",
    "Microsoft.Network/*/read"
  ],
  "NotActions": [
    "Microsoft.Network/networkSecurityGroups/delete"
  ],
  "AssignableScopes": ["/subscriptions/{sub-id}/resourceGroups/{rg-name}"]
}
```

### Updating Custom Roles

```powershell
# Get existing custom role
$role = Get-AzRoleDefinition "Virtual Machine Operator"

# Add new action
$role.Actions.Add("Microsoft.Compute/virtualMachines/powerOff/action")

# Update role
Set-AzRoleDefinition -Role $role
```

### Deleting Custom Roles

```powershell
# Remove custom role
Remove-AzRoleDefinition -Name "Virtual Machine Operator"
```

```bash
# Azure CLI
az role definition delete --name "Virtual Machine Operator"
```

---

## Role Assignments

### Assigning Roles

#### Azure Portal
1. Navigate to the **scope** (subscription, resource group, or resource)
2. Select **Access control (IAM)**
3. Click **+ Add** > **Add role assignment**
4. **Role tab**: Select the role
5. **Members tab**: Select members (users, groups, service principals)
6. **Review + assign**

#### PowerShell
```powershell
# Assign role at subscription scope
New-AzRoleAssignment `
    -SignInName "user@contoso.com" `
    -RoleDefinitionName "Reader" `
    -Scope "/subscriptions/{subscription-id}"

# Assign role at resource group scope
New-AzRoleAssignment `
    -ObjectId "00000000-0000-0000-0000-000000000000" `
    -RoleDefinitionName "Contributor" `
    -ResourceGroupName "MyResourceGroup"

# Assign role at resource scope
New-AzRoleAssignment `
    -SignInName "user@contoso.com" `
    -RoleDefinitionName "Virtual Machine Contributor" `
    -ResourceName "myVM" `
    -ResourceType "Microsoft.Compute/virtualMachines" `
    -ResourceGroupName "MyResourceGroup"

# Assign role to a group
New-AzRoleAssignment `
    -ObjectId "22222222-2222-2222-2222-222222222222" `
    -RoleDefinitionName "Contributor" `
    -ResourceGroupName "MyResourceGroup"
```

#### Azure CLI
```bash
# Assign role at subscription scope
az role assignment create \
    --assignee "user@contoso.com" \
    --role "Reader" \
    --scope "/subscriptions/{subscription-id}"

# Assign role at resource group scope
az role assignment create \
    --assignee-object-id "00000000-0000-0000-0000-000000000000" \
    --role "Contributor" \
    --resource-group "MyResourceGroup"

# Assign custom role
az role assignment create \
    --assignee "user@contoso.com" \
    --role "Virtual Machine Operator" \
    --resource-group "MyResourceGroup"
```

### Listing Role Assignments

```powershell
# List all role assignments for a subscription
Get-AzRoleAssignment -Scope "/subscriptions/{subscription-id}"

# List role assignments for a specific user
Get-AzRoleAssignment -SignInName "user@contoso.com"

# List role assignments for a resource group
Get-AzRoleAssignment -ResourceGroupName "MyResourceGroup"

# List role assignments including inherited
Get-AzRoleAssignment -ResourceGroupName "MyResourceGroup" -IncludeInherited
```

```bash
# Azure CLI
az role assignment list --scope "/subscriptions/{subscription-id}"
az role assignment list --assignee "user@contoso.com"
az role assignment list --resource-group "MyResourceGroup"
```

### Removing Role Assignments

```powershell
# Remove role assignment
Remove-AzRoleAssignment `
    -SignInName "user@contoso.com" `
    -RoleDefinitionName "Reader" `
    -ResourceGroupName "MyResourceGroup"
```

```bash
# Azure CLI
az role assignment delete \
    --assignee "user@contoso.com" \
    --role "Reader" \
    --resource-group "MyResourceGroup"
```

---

## Scope Hierarchy

RBAC uses a hierarchical scope model. Permissions assigned at a parent scope are inherited by child scopes.

```
Management Group (optional)
    └── Subscription
            └── Resource Group
                    └── Resource
```

### Inheritance Rules

1. **Role assignments are additive** across scopes
2. **Child scopes inherit** permissions from parent scopes
3. **More specific assignments** don't override parent assignments
4. **Deny assignments** override allow assignments at all levels

### Example Inheritance Scenario

**User has:**
- **Reader** at Subscription scope
- **Contributor** at Resource Group "RG-Prod" scope

**Result:**
- User can **view** all resources in subscription (inherited Reader)
- User can **modify** resources in RG-Prod (explicit Contributor)
- User can **view but not modify** resources in other resource groups

### Scopes in Practice

#### Management Group Scope
```powershell
New-AzRoleAssignment `
    -SignInName "user@contoso.com" `
    -RoleDefinitionName "Reader" `
    -Scope "/providers/Microsoft.Management/managementGroups/MyManagementGroup"
```

#### Subscription Scope
```powershell
New-AzRoleAssignment `
    -SignInName "user@contoso.com" `
    -RoleDefinitionName "Contributor" `
    -Scope "/subscriptions/00000000-0000-0000-0000-000000000000"
```

#### Resource Group Scope
```powershell
New-AzRoleAssignment `
    -SignInName "user@contoso.com" `
    -RoleDefinitionName "Contributor" `
    -ResourceGroupName "MyResourceGroup"
```

#### Resource Scope
```powershell
New-AzRoleAssignment `
    -SignInName "user@contoso.com" `
    -RoleDefinitionName "Virtual Machine Contributor" `
    -ResourceName "myVM" `
    -ResourceType "Microsoft.Compute/virtualMachines" `
    -ResourceGroupName "MyResourceGroup"
```

---

## Interpreting Access Assignments

### Check Access Feature

#### Azure Portal
1. Navigate to resource/resource group/subscription
2. Select **Access control (IAM)**
3. Click **Check access**
4. Search for user, group, or service principal
5. View assigned roles and inherited permissions

### Understanding Effective Permissions

**Scenario:** User John has the following assignments:
- **Reader** at Subscription scope
- **Contributor** at Resource Group "RG-Web" scope
- **Owner** at VM "WebServer1" (in RG-Web)

**Effective Permissions:**
| Scope | Role | Can View | Can Modify | Can Assign Roles |
|-------|------|----------|------------|------------------|
| Subscription | Reader | ✓ | ✗ | ✗ |
| RG-Web | Contributor | ✓ | ✓ | ✗ |
| WebServer1 VM | Owner | ✓ | ✓ | ✓ |
| Other RGs | Reader (inherited) | ✓ | ✗ | ✗ |

### Deny Assignments

**Deny assignments** block users from performing specific actions even if a role assignment grants them access.

- Created by Azure (for system protection)
- Cannot be created directly by users
- **Override all allow assignments**
- Rare in normal scenarios

#### Viewing Deny Assignments
```powershell
Get-AzDenyAssignment -Scope "/subscriptions/{subscription-id}"
```

### Troubleshooting Access

**Common issues:**

1. **User can't access resource**
   - Check role assignments at all scope levels
   - Verify role definition includes required actions
   - Check for deny assignments
   - Verify user's Microsoft Entra ID status

2. **User has too much access**
   - Review inherited permissions from parent scopes
   - Check group memberships (roles assigned to groups)
   - Review custom role definitions

3. **Role assignment not taking effect**
   - Wait for propagation (can take up to 5 minutes)
   - Sign out and sign back in
   - Check if conditional access policies are blocking

---

## PowerShell and CLI Commands

### Essential PowerShell Commands

```powershell
# Role Definitions
Get-AzRoleDefinition
Get-AzRoleDefinition "Contributor"
Get-AzRoleDefinition | Where-Object {$_.IsCustom -eq $true}
New-AzRoleDefinition -InputFile "role.json"
Set-AzRoleDefinition -Role $roleObject
Remove-AzRoleDefinition -Name "Custom Role Name"

# Role Assignments
New-AzRoleAssignment -SignInName "user@contoso.com" -RoleDefinitionName "Reader" -Scope "/subscriptions/{sub-id}"
Get-AzRoleAssignment
Get-AzRoleAssignment -SignInName "user@contoso.com"
Get-AzRoleAssignment -ResourceGroupName "MyRG"
Get-AzRoleAssignment -ObjectId "guid" -ExpandPrincipalGroups
Remove-AzRoleAssignment -SignInName "user@contoso.com" -RoleDefinitionName "Reader" -Scope "..."

# Check access
Get-AzRoleAssignment -SignInName "user@contoso.com" -IncludeClassicAdministrators
```

### Essential Azure CLI Commands

```bash
# Role Definitions
az role definition list
az role definition list --name "Contributor"
az role definition list --custom-role-only true
az role definition create --role-definition role.json
az role definition update --role-definition role.json
az role definition delete --name "Custom Role Name"

# Role Assignments
az role assignment create --assignee "user@contoso.com" --role "Reader" --scope "..."
az role assignment list
az role assignment list --assignee "user@contoso.com"
az role assignment list --resource-group "MyRG"
az role assignment list --all
az role assignment delete --assignee "user@contoso.com" --role "Reader"
```

---

## Best Practices

### General RBAC Best Practices

1. **Principle of Least Privilege**
   - Grant minimum permissions necessary
   - Avoid Owner role unless absolutely needed
   - Use Reader role for view-only access

2. **Use Groups for Assignments**
   - Assign roles to groups, not individual users
   - Easier to manage and audit
   - Reduces number of role assignments

3. **Leverage Built-in Roles**
   - Use built-in roles when possible
   - Create custom roles only when necessary
   - Document custom role purposes

4. **Scope Appropriately**
   - Assign roles at the narrowest scope needed
   - Avoid subscription-level assignments unless required
   - Consider resource group or resource-level assignments

5. **Regular Access Reviews**
   - Periodically review role assignments
   - Remove unnecessary permissions
   - Audit custom roles and update as needed

### Custom Role Best Practices

1. **Clear Naming Conventions**
   - Use descriptive, consistent names
   - Include purpose in description

2. **Minimize Wildcards**
   - Be specific with Actions
   - Avoid `*` unless necessary

3. **Version Control**
   - Store role definitions in source control
   - Track changes over time
   - Use comments in JSON

4. **Test Before Production**
   - Test custom roles in non-prod environment
   - Verify all required permissions
   - Check for unintended permissions

5. **Document Use Cases**
   - Document why custom role was created
   - List intended users/scenarios
   - Maintain updated documentation

### Security Best Practices

1. **Separate Admin Accounts**
   - Use different accounts for admin tasks
   - Apply Privileged Identity Management (PIM)
   - Enable MFA for admin accounts

2. **Avoid Classic Administrators**
   - Migrate from classic administrators to RBAC
   - Classic roles are legacy and less granular

3. **Monitor Role Changes**
   - Set up alerts for role assignment changes
   - Review audit logs regularly
   - Use Azure Policy for governance

4. **Conditional Access**
   - Combine RBAC with Conditional Access policies
   - Require MFA for sensitive roles
   - Restrict access by location or device

---

## Exam Tips

### Key Concepts to Remember

1. **RBAC vs Entra ID Roles**: Understand the difference
2. **Fundamental Roles**: Owner, Contributor, Reader, User Access Administrator
3. **Custom Roles**: When and how to create them
4. **Scope Hierarchy**: Management Group > Subscription > Resource Group > Resource
5. **Inheritance**: Child scopes inherit from parent scopes
6. **Role Assignment Formula**: Principal + Role + Scope

### Common Exam Scenarios

1. **Grant least privilege access**: Choose the most restrictive role that allows required actions
2. **Delegating administrative tasks**: Use appropriate built-in roles or create custom roles
3. **Troubleshooting access issues**: Check role assignments at all scope levels
4. **Restricting specific actions**: Use NotActions in custom roles
5. **Managing multiple subscriptions**: Use management groups for role assignment

### Important Differences

| Owner | Contributor |
|-------|-------------|
| Can assign roles | Cannot assign roles |
| Full access to all resources | Full access except role management |

| Actions | DataActions |
|---------|-------------|
| Management plane operations | Data plane operations |
| Create/delete resources | Read/write data in resources |
| Examples: Create VM, Delete Storage Account | Examples: Read blob, Write to table |

### Commands to Memorize

```powershell
# Most common PowerShell commands
Get-AzRoleDefinition
New-AzRoleDefinition -InputFile "role.json"
New-AzRoleAssignment -SignInName "..." -RoleDefinitionName "..." -Scope "..."
Get-AzRoleAssignment
Remove-AzRoleAssignment
```

```bash
# Most common CLI commands
az role definition list
az role definition create --role-definition role.json
az role assignment create --assignee "..." --role "..." --scope "..."
az role assignment list
az role assignment delete
```

### Watch Out For

- **Actions vs DataActions**: Management plane vs data plane
- **NotActions**: Subtracts from Actions (not an explicit deny)
- **Scope format**: Correct format is `/subscriptions/{id}/resourceGroups/{name}`
- **Custom role limits**: Max 5000 custom roles per directory
- **Assignment propagation**: Can take up to 5 minutes
- **ObjectId vs SignInName**: Use ObjectId for service principals

### Troubleshooting Questions

1. **User can't perform action**: Check effective permissions at all scope levels
2. **Too many role assignments**: Consolidate using groups
3. **Custom role not working**: Verify Actions, scope, and assignment
4. **Inherited permissions**: Understand parent-to-child inheritance
5. **Deny assignments**: Remember they override all allows

---

## Related Topics
- [Manage AAD Objects](./Manage%20AAD%20Objects.md) - User and group management
- [Manage Subscriptions and Governance](./Manage%20Subscriptions%20and%20Governance.md)
- [Official RBAC Documentation](https://learn.microsoft.com/en-us/azure/role-based-access-control/)

---

**Next Steps:**
1. Practice assigning built-in roles at different scopes
2. Create and test a custom role
3. Review role assignments for a resource using Check Access
4. Practice interpreting effective permissions
5. Set up alerts for role assignment changes
