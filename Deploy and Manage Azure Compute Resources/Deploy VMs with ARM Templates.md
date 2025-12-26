# Deploy VMs with ARM Templates - Azure Administrator Exam Prep (AZ-104)

## Table of Contents
1. [ARM Templates Overview](#arm-templates-overview)
2. [Template Structure and Syntax](#template-structure-and-syntax)
3. [Parameters, Variables, and Functions](#parameters-variables-and-functions)
4. [Template Deployment Methods](#template-deployment-methods)
5. [Bicep Infrastructure as Code](#bicep-infrastructure-as-code)
6. [Linked and Nested Templates](#linked-and-nested-templates)
7. [Template Validation and What-If](#template-validation-and-what-if)
8. [Exporting and Reusing Templates](#exporting-and-reusing-templates)
9. [PowerShell & CLI Commands](#powershell--cli-commands)
10. [Exam Tips & Key Concepts](#exam-tips--key-concepts)
11. [Exam Scenarios](#exam-scenarios)
12. [Troubleshooting Guide](#troubleshooting-guide)

---

## ARM Templates Overview

### What Are ARM Templates?

Azure Resource Manager (ARM) templates are JSON files that define infrastructure as code (IaC) for consistent, repeatable Azure deployments.

**Key Characteristics:**
- **Declarative Syntax**: Describe desired state, not sequence of steps
- **Idempotent**: Deploy same template multiple times with same results
- **Repeatable**: Consistent deployments across environments
- **Versioned**: Track infrastructure changes like code
- **Orchestration**: Resource Manager handles deployment order
- **Parallel Deployment**: Deploy independent resources concurrently

**Why Choose ARM Templates?**

1. **Entire Infrastructure in One File**: VMs, networks, storage, and all dependencies
2. **Infrastructure as Code**: Track changes, manage versions, integrate with CI/CD
3. **Modular**: Break complex deployments into linked templates
4. **Azure-Native**: Deploy resources immediately when available
5. **Exportable**: Export existing resources to templates
6. **Extensible**: Add custom logic with deployment scripts
7. **No Tool Updates**: New Azure features work immediately

### Template Design Approaches

**Single Template Approach:**
- All resources in one file
- Best for small, simple deployments (< 50 resources)
- Easy to understand and maintain
- No external dependencies

**Nested Templates (Embedded):**
- Secondary templates embedded within main template
- Templates deployed inline
- Template syntax embedded as JSON strings
- Limited reusability

**Linked Templates (Recommended):**
- Separate template files referenced from main template
- Modularity and reusability
- Each template maintains separate version
- Supports team development
- Can be stored in git, Azure Storage, or CDN
- Best for complex deployments

**Template Specs:**
- Templates stored as resources in Azure
- RBAC access control
- Versioning and lifecycle management
- Ideal for enterprise distribution

---

## Template Structure and Syntax

### Template Format

Basic ARM template JSON structure:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "metadata": {
    "description": "Deploys a VM with networking and storage"
  },
  "parameters": {},
  "variables": {},
  "functions": [],
  "resources": [],
  "outputs": {}
}
```

### Template Elements

| Element | Required | Purpose | Example |
|---------|----------|---------|---------|
| **$schema** | Yes | JSON schema version URL | `2019-04-01` for resource group deployments |
| **contentVersion** | Yes | Template version (document significance) | `1.0.0.0` |
| **metadata** | No | Template description, author, etc. | Comments and documentation |
| **parameters** | No | Input values at deployment time | VM size, admin username, location |
| **variables** | No | Values reused throughout template | Storage account name, VNET address space |
| **functions** | No | User-defined functions | Custom name generation, complex logic |
| **resources** | Yes | Azure resources to deploy | VMs, networks, disks, etc. |
| **outputs** | No | Return values after deployment | VM IP address, resource IDs |

### Schema Versions by Deployment Scope

| Scope | Schema URL |
|-------|-----------|
| Resource Group | `https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#` |
| Subscription | `https://schema.management.azure.com/schemas/2018-05-01/subscriptionDeploymentTemplate.json#` |
| Management Group | `https://schema.management.azure.com/schemas/2019-08-01/managementGroupDeploymentTemplate.json#` |
| Tenant | `https://schema.management.azure.com/schemas/2019-08-01/tenantDeploymentTemplate.json#` |

**Use latest schema for Visual Studio Code:** `2019-04-01` includes all features for resource group deployments.

### Language Version 2.0 (Enhanced Syntax)

Add `languageVersion: "2.0"` for advanced features:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "languageVersion": "2.0",
  "contentVersion": "1.0.0.0",
  "resources": {
    "storageAccount": {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2021-04-01",
      "name": "[uniqueString(resourceGroup().id)]",
      "location": "eastus",
      "sku": { "name": "Standard_LRS" },
      "kind": "StorageV2",
      "properties": { "accessTier": "Hot" }
    }
  }
}
```

**languageVersion 2.0 Features:**
- Symbolic names for resources (instead of arrays)
- Use resource names in copy loops and dependsOn
- New references() function for arrays
- User-defined types
- Simplified syntax

---

## Parameters, Variables, and Functions

### Parameters

Input values provided at deployment time for customization.

```json
"parameters": {
  "vmSize": {
    "type": "string",
    "defaultValue": "Standard_D2s_v3",
    "metadata": {
      "description": "Size of the virtual machine"
    },
    "allowedValues": [
      "Standard_B2s",
      "Standard_D2s_v3",
      "Standard_E4s_v5"
    ]
  },
  "adminUsername": {
    "type": "string",
    "metadata": {
      "description": "Administrator username"
    }
  },
  "adminPassword": {
    "type": "securestring",
    "metadata": {
      "description": "Administrator password (min 12 chars)"
    }
  },
  "location": {
    "type": "string",
    "defaultValue": "[resourceGroup().location]",
    "metadata": {
      "description": "Location for resources"
    }
  }
}
```

**Parameter Types:**
- `string` - Text values
- `int` - Numeric values
- `bool` - True/false values
- `object` - Complex JSON structures
- `array` - Multiple values
- `securestring` - Passwords, keys (not shown in history)
- `secureObject` - Complex sensitive data

**Best Practices:**
- Use `securestring` for passwords and keys
- Provide default values when possible
- Use `allowedValues` to restrict choices
- Add metadata descriptions (show in portal)
- Limit to 256 parameters

### Variables

Simplify templates by defining reusable values:

```json
"variables": {
  "vmName": "[concat('vm-', uniqueString(resourceGroup().id))]",
  "storageName": "[concat(toLower(parameters('storageNamePrefix')), uniqueString(resourceGroup().id))]",
  "vnetName": "myVNet",
  "subnetName": "mySubnet",
  "nicName": "[concat(variables('vmName'), '-nic')]",
  "vnetAddressPrefix": "10.0.0.0/16",
  "subnetAddressPrefix": "10.0.0.0/24",
  "environment": {
    "prod": {
      "vmSize": "Standard_E4s_v5",
      "storageType": "Premium_LRS"
    },
    "dev": {
      "vmSize": "Standard_B2s",
      "storageType": "Standard_LRS"
    }
  },
  "selectedEnvironment": "[variables('environment')[parameters('environmentType')]]"
}
```

**Variable Construction:**
- Use functions like `concat()`, `toUpper()`, `toLower()`
- Reference other variables: `"[variables('vmName')]"`
- Reference parameters: `"[parameters('location')]"`
- Cannot use `reference()` or `list()` functions (runtime only)

**Best Practices:**
- Use for values referenced multiple times
- Use for complex expressions
- Use camelCase for variable names
- Remove unused variables
- Limit to 256 variables

### Template Functions

Built-in functions extend template capabilities:

**String Functions:**
- `concat()` - Join strings
- `toUpper()` / `toLower()` - Case conversion
- `substring()` - Extract substring
- `replace()` - Find and replace
- `split()` - Split by delimiter
- `length()` - String length
- `uri()` - Construct URIs

**Resource Functions:**
- `reference()` - Get runtime state of resource
- `resourceId()` - Get full resource ID
- `subscription()` - Subscription info
- `resourceGroup()` - Resource group info

**Utility Functions:**
- `uniqueString()` - Generate unique string
- `base64()` / `base64ToJson()` - Encoding
- `json()` - Parse JSON strings
- `copy()` - Iterate collections
- `if()` - Conditional logic

**User-Defined Functions:**

```json
"functions": [
  {
    "namespace": "contoso",
    "members": {
      "uniqueName": {
        "parameters": [
          {
            "name": "namePrefix",
            "type": "string"
          }
        ],
        "output": {
          "type": "string",
          "value": "[concat(toLower(parameters('namePrefix')), uniqueString(resourceGroup().id))]"
        }
      }
    }
  }
],
"resources": [
  {
    "name": "[contoso.uniqueName('vm')]",
    "type": "Microsoft.Compute/virtualMachines"
  }
]
```

---

## Template Deployment Methods

### Deployment Scopes

**Resource Group:**
- Deploy to existing resource group
- Most common for VMs and resources
- Create resource group first: `New-AzResourceGroup`

**Subscription:**
- Deploy across resource groups
- Create management resources
- Policy assignments, role assignments

**Management Group:**
- Deploy across subscriptions
- Enterprise-wide policies
- Governance structures

### PowerShell Deployment

**Create Resource Group:**

```powershell
New-AzResourceGroup `
    -Name "myResourceGroup" `
    -Location "East US"
```

**Deploy Template (Local File):**

```powershell
New-AzResourceGroupDeployment `
    -Name "vmDeployment" `
    -ResourceGroupName "myResourceGroup" `
    -TemplateFile "./azuredeploy.json" `
    -vmSize "Standard_D2s_v3" `
    -adminUsername "azureuser" `
    -adminPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force)
```

**Deploy Template (URI):**

```powershell
New-AzResourceGroupDeployment `
    -Name "vmDeployment" `
    -ResourceGroupName "myResourceGroup" `
    -TemplateUri "https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/quickstarts/microsoft.compute/vm-windows/azuredeploy.json" `
    -adminUsername "azureuser" `
    -adminPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force)
```

**Deploy with Parameters File:**

```powershell
New-AzResourceGroupDeployment `
    -Name "vmDeployment" `
    -ResourceGroupName "myResourceGroup" `
    -TemplateFile "./azuredeploy.json" `
    -TemplateParameterFile "./azuredeploy.parameters.json"
```

**Deploy at Subscription Scope:**

```powershell
New-AzSubscriptionDeployment `
    -Location "East US" `
    -Name "subscriptionDeployment" `
    -TemplateFile "./azuredeploy.json"
```

### Azure CLI Deployment

**Create Resource Group:**

```bash
az group create \
    --name myResourceGroup \
    --location eastus
```

**Deploy Template (Local File):**

```bash
az deployment group create \
    --name vmDeployment \
    --resource-group myResourceGroup \
    --template-file azuredeploy.json \
    --parameters vmSize=Standard_D2s_v3 adminUsername=azureuser adminPassword=P@ssw0rd123!
```

**Deploy Template (URI):**

```bash
az deployment group create \
    --name vmDeployment \
    --resource-group myResourceGroup \
    --template-uri "https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/quickstarts/microsoft.compute/vm-windows/azuredeploy.json" \
    --parameters adminUsername=azureuser adminPassword=P@ssw0rd123!
```

**Deploy with Parameters File:**

```bash
az deployment group create \
    --name vmDeployment \
    --resource-group myResourceGroup \
    --template-file azuredeploy.json \
    --parameters @azuredeploy.parameters.json
```

**Deploy at Subscription Scope:**

```bash
az deployment sub create \
    --location eastus \
    --name subscriptionDeployment \
    --template-file azuredeploy.json
```

### Deployment Modes

**Incremental (Default):**
- Add/update specified resources
- Leave other resources unchanged
- Safe for partial updates
- Recommended for most scenarios

**Complete:**
- Deploy all resources in template
- Delete resources not in template
- Useful for isolated resource groups
- Risk of unintended deletion

```powershell
# Incremental deployment
New-AzResourceGroupDeployment `
    -Mode Incremental `
    -ResourceGroupName "myResourceGroup" `
    -TemplateFile "./azuredeploy.json"

# Complete deployment (deletes resources not in template)
New-AzResourceGroupDeployment `
    -Mode Complete `
    -ResourceGroupName "myResourceGroup" `
    -TemplateFile "./azuredeploy.json"
```

---

## Bicep Infrastructure as Code

### Bicep Overview

Bicep is a domain-specific language that simplifies ARM templates.

**Bicep vs ARM JSON:**

| Feature | ARM JSON | Bicep |
|---------|----------|-------|
| **Syntax** | Verbose JSON | Readable language |
| **Learning Curve** | Steep | Gentle |
| **File Size** | Larger | Compact |
| **Intellisense** | Limited | Excellent |
| **Reusability** | Linked templates | Modules |
| **Recommended** | Legacy | New deployments |

**Bicep Advantages:**
- Cleaner, more readable syntax
- No escape characters or complex nesting
- Better tooling and validation
- Automatic dependency resolution
- Simplified loops and conditionals
- Import external modules
- Built-in functions
- Type safety

### Bicep Template Example

```bicep
param vmName string = 'myVM'
param vmSize string = 'Standard_D2s_v3'
param adminUsername string
@secure()
param adminPassword string
param location string = resourceGroup().location
param environment string = 'dev'

// Variables
var storageName = 'st${uniqueString(resourceGroup().id)}'
var vnetName = 'vnet-${vmName}'
var subnetName = 'subnet-${vmName}'
var nicName = 'nic-${vmName}'
var nsgName = 'nsg-${vmName}'

// Conditionals
var vmTags = environment == 'prod' ? {
  environment: 'production'
  costCenter: '12345'
} : {
  environment: 'development'
  costCenter: '67890'
}

// Resources
resource storageAccount 'Microsoft.Storage/storageAccounts@2021-04-01' = {
  name: storageName
  location: location
  kind: 'StorageV2'
  sku: {
    name: 'Standard_LRS'
  }
  properties: {
    accessTier: 'Hot'
  }
}

resource vnet 'Microsoft.Network/virtualNetworks@2021-02-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/16'
      ]
    }
    subnets: [
      {
        name: subnetName
        properties: {
          addressPrefix: '10.0.0.0/24'
          networkSecurityGroup: {
            id: nsg.id
          }
        }
      }
    ]
  }
}

resource nsg 'Microsoft.Network/networkSecurityGroups@2021-02-01' = {
  name: nsgName
  location: location
  properties: {
    securityRules: [
      {
        name: 'AllowRDP'
        properties: {
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '3389'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
          access: 'Allow'
          priority: 1000
          direction: 'Inbound'
        }
      }
    ]
  }
}

resource nic 'Microsoft.Network/networkInterfaces@2021-02-01' = {
  name: nicName
  location: location
  properties: {
    ipConfigurations: [
      {
        name: 'ipconfig1'
        properties: {
          subnet: {
            id: '${vnet.id}/subnets/${subnetName}'
          }
          privateIPAllocationMethod: 'Dynamic'
        }
      }
    ]
    networkSecurityGroup: {
      id: nsg.id
    }
  }
}

resource vm 'Microsoft.Compute/virtualMachines@2021-03-01' = {
  name: vmName
  location: location
  tags: vmTags
  properties: {
    hardwareProfile: {
      vmSize: vmSize
    }
    osProfile: {
      computerName: vmName
      adminUsername: adminUsername
      adminPassword: adminPassword
    }
    storageProfile: {
      imageReference: {
        publisher: 'MicrosoftWindowsServer'
        offer: 'WindowsServer'
        sku: '2019-Datacenter'
        version: 'latest'
      }
      osDisk: {
        createOption: 'FromImage'
        managedDisk: {
          storageAccountType: 'Premium_LRS'
        }
      }
    }
    networkProfile: {
      networkInterfaces: [
        {
          id: nic.id
          properties: {
            primary: true
          }
        }
      ]
    }
  }
}

// Outputs
output vmId string = vm.id
output vmName string = vm.name
output storageId string = storageAccount.id
output vnetId string = vnet.id
```

### Bicep Deployment

```powershell
# Deploy Bicep file
New-AzResourceGroup -Name exampleRG -Location eastus
New-AzResourceGroupDeployment `
    -ResourceGroupName exampleRG `
    -TemplateFile ./main.bicep `
    -vmName "myVM" `
    -adminUsername "azureuser" `
    -adminPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force)
```

```bash
# Deploy Bicep file with Azure CLI
az group create --name exampleRG --location eastus
az deployment group create \
    --resource-group exampleRG \
    --template-file main.bicep \
    --parameters vmName=myVM adminUsername=azureuser adminPassword=P@ssw0rd123!
```

---

## Linked and Nested Templates

### Linked Templates (Recommended)

Deploy separate template files referenced from main template:

**Main Template (main.json):**

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "templateUri": {
      "type": "string",
      "defaultValue": "https://raw.githubusercontent.com/myrepo/templates/"
    }
  },
  "resources": [
    {
      "type": "Microsoft.Resources/deployments",
      "apiVersion": "2020-10-01",
      "name": "storageDeployment",
      "properties": {
        "mode": "Incremental",
        "templateLink": {
          "uri": "[concat(parameters('templateUri'), 'storage.json')]",
          "contentVersion": "1.0.0.0"
        },
        "parameters": {
          "storageAccountName": {
            "value": "mystorageaccount"
          },
          "location": {
            "value": "[resourceGroup().location]"
          }
        }
      }
    },
    {
      "type": "Microsoft.Resources/deployments",
      "apiVersion": "2020-10-01",
      "name": "vmDeployment",
      "dependsOn": [
        "storageDeployment"
      ],
      "properties": {
        "mode": "Incremental",
        "templateLink": {
          "uri": "[concat(parameters('templateUri'), 'vm.json')]"
        },
        "parameters": {
          "vmName": {
            "value": "myVM"
          },
          "storageId": {
            "value": "[reference('storageDeployment').outputs.storageId.value]"
          }
        }
      }
    }
  ],
  "outputs": {
    "storageOutput": {
      "type": "object",
      "value": "[reference('storageDeployment').outputs]"
    },
    "vmOutput": {
      "type": "object",
      "value": "[reference('vmDeployment').outputs]"
    }
  }
}
```

**Linked Template (storage.json):**

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": {
      "type": "string"
    },
    "location": {
      "type": "string"
    }
  },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2021-04-01",
      "name": "[parameters('storageAccountName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "Standard_LRS"
      },
      "kind": "StorageV2",
      "properties": {
        "accessTier": "Hot"
      }
    }
  ],
  "outputs": {
    "storageId": {
      "type": "string",
      "value": "[resourceId('Microsoft.Storage/storageAccounts', parameters('storageAccountName'))]"
    },
    "storageEndpoint": {
      "type": "string",
      "value": "[reference(parameters('storageAccountName')).primaryEndpoints.blob]"
    }
  }
}
```

### Nested Templates (Embedded)

Embed template syntax directly in main template:

```json
{
  "type": "Microsoft.Resources/deployments",
  "apiVersion": "2020-10-01",
  "name": "nestedTemplate",
  "properties": {
    "mode": "Incremental",
    "template": {
      "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
      "contentVersion": "1.0.0.0",
      "resources": [
        {
          "type": "Microsoft.Storage/storageAccounts",
          "apiVersion": "2021-04-01",
          "name": "[uniqueString(resourceGroup().id)]",
          "location": "[resourceGroup().location]",
          "sku": {
            "name": "Standard_LRS"
          },
          "kind": "StorageV2"
        }
      ]
    }
  }
}
```

### Bicep Modules

Bicep simplifies linked templates with modules:

**Main Bicep (main.bicep):**

```bicep
param location string = resourceGroup().location
param vmName string = 'myVM'

module storage 'modules/storage.bicep' = {
  name: 'storageModule'
  params: {
    location: location
    storageNamePrefix: 'st'
  }
}

module vm 'modules/vm.bicep' = {
  name: 'vmModule'
  params: {
    vmName: vmName
    location: location
    storageAccountId: storage.outputs.storageId
  }
}

output storageId string = storage.outputs.storageId
output vmId string = vm.outputs.vmId
```

**Storage Module (modules/storage.bicep):**

```bicep
param location string
param storageNamePrefix string

resource storageAccount 'Microsoft.Storage/storageAccounts@2021-04-01' = {
  name: '${storageNamePrefix}${uniqueString(resourceGroup().id)}'
  location: location
  kind: 'StorageV2'
  sku: {
    name: 'Standard_LRS'
  }
}

output storageId string = storageAccount.id
output storageName string = storageAccount.name
```

---

## Template Validation and What-If

### Validate Template

Check syntax before deployment:

**PowerShell Validation:**

```powershell
# Test template syntax
Test-AzResourceGroupDeployment `
    -ResourceGroupName "myResourceGroup" `
    -TemplateFile "./azuredeploy.json"

# Returns: ProvisioningState = Succeeded (if valid)
```

**Azure CLI Validation:**

```bash
# Validate template
az deployment group validate \
    --resource-group myResourceGroup \
    --template-file azuredeploy.json \
    --parameters @azuredeploy.parameters.json
```

### What-If Preview

Preview deployment changes without applying:

```powershell
# Preview changes (what will be created/modified/deleted)
New-AzResourceGroupDeployment `
    -ResourceGroupName "myResourceGroup" `
    -TemplateFile "./azuredeploy.json" `
    -WhatIf
```

**What-If Results:**
- **Create**: New resources
- **Modify**: Changed properties
- **Ignore**: Unchanged resources
- **Delete**: Removed resources (Complete mode only)

---

## Exporting and Reusing Templates

### Export Existing Resources

**From Azure Portal:**

1. Go to resource group
2. Click **Deployments** → Select deployment
3. Click **Template** → Download

**From PowerShell:**

```powershell
# Export from resource group
Save-AzResourceGroupTemplate `
    -ResourceGroupName "myResourceGroup" `
    -Path "./exported-template"

# Export from deployment history
(Get-AzResourceGroupDeployment `
    -ResourceGroupName "myResourceGroup" `
    -Name "myDeployment").Template | ConvertTo-Json | Out-File "template.json"
```

**From Azure CLI:**

```bash
# Export from resource group
az group export \
    --name myResourceGroup \
    --output-format json > template.json

# Export from deployment
az deployment group export \
    --resource-group myResourceGroup \
    --name myDeployment > template.json
```

### Export Limitations

- **Not guaranteed**: Export may fail for some resources
- **Incomplete parameters**: Properties often hardcoded
- **Requires cleanup**: Remove unnecessary properties before reuse
- **Hand-written templates better**: Create from scratch for production

### Template Reusability

**Using Template Specs:**

```powershell
# Create template spec
New-AzTemplateSpec `
    -ResourceGroupName "templateSpecsRG" `
    -Name "vmTemplate" `
    -DisplayName "VM Deployment Template" `
    -Description "Deploy Windows VM with networking" `
    -Version "1.0" `
    -TemplateFile "./azuredeploy.json"

# Deploy template spec
New-AzResourceGroupDeployment `
    -ResourceGroupName "myResourceGroup" `
    -TemplateSpecId "/subscriptions/xxx/resourceGroups/templateSpecsRG/providers/Microsoft.Resources/templateSpecs/vmTemplate/versions/1.0"
```

---

## PowerShell & CLI Commands

### Template Operations

```powershell
# Validate template
Test-AzResourceGroupDeployment -ResourceGroupName "rg" -TemplateFile "template.json"

# Deploy template
New-AzResourceGroupDeployment -ResourceGroupName "rg" -TemplateFile "template.json"

# Deploy with parameters
New-AzResourceGroupDeployment -ResourceGroupName "rg" `
    -TemplateFile "template.json" `
    -TemplateParameterFile "parameters.json"

# Deploy with inline parameters
New-AzResourceGroupDeployment -ResourceGroupName "rg" `
    -TemplateFile "template.json" `
    -vmSize "Standard_D2s_v3" `
    -adminUsername "azureuser"

# Deploy with what-if preview
New-AzResourceGroupDeployment -ResourceGroupName "rg" `
    -TemplateFile "template.json" -WhatIf

# Get deployment status
Get-AzResourceGroupDeployment -ResourceGroupName "rg" -Name "deploymentName"

# Get deployment outputs
(Get-AzResourceGroupDeployment -ResourceGroupName "rg" -Name "deploymentName").Outputs

# Export template from resource group
Save-AzResourceGroupTemplate -ResourceGroupName "rg" -Path "./"

# Create template spec
New-AzTemplateSpec -ResourceGroupName "rg" -Name "spec" -Version "1.0" -TemplateFile "template.json"
```

```bash
# Validate template
az deployment group validate --resource-group rg --template-file template.json

# Deploy template
az deployment group create --resource-group rg --template-file template.json

# Deploy with parameters
az deployment group create --resource-group rg \
    --template-file template.json \
    --parameters @parameters.json

# Deploy with inline parameters
az deployment group create --resource-group rg \
    --template-file template.json \
    --parameters vmSize=Standard_D2s_v3 adminUsername=azureuser

# Preview changes
az deployment group what-if --resource-group rg --template-file template.json

# Get deployment info
az deployment group show --resource-group rg --name deploymentName

# Export template
az group export --name rg > template.json

# Create template spec
az ts create --resource-group rg --name "spec" --version "1.0" --template-file template.json
```

---

## Exam Tips & Key Concepts

### Critical Concepts for AZ-104

1. **Template Structure**
   - Always include `$schema` and `contentVersion`
   - Resources array required (even if empty)
   - Parameters for customization, variables for reusability
   - Outputs for returning values

2. **Parameters**
   - Use `securestring` for passwords/keys (not shown in history)
   - Provide reasonable defaults
   - Use `allowedValues` to restrict choices
   - Limited to 256 parameters per template

3. **Variables**
   - Cannot use `reference()` or `list()` functions
   - Use functions like `concat()`, `uniqueString()`
   - Cannot reference parameters in variable declarations
   - Best for complex expressions used multiple times

4. **Deployment Methods**
   - Incremental: Default, safe, adds/updates only specified resources
   - Complete: Deletes resources not in template (risky)
   - Always validate before deploying to production
   - Use what-if to preview changes

5. **Linked vs Nested**
   - Linked: Separate files, better modularity, reusable
   - Nested: Embedded JSON, less reusable, simpler for small projects
   - Linked templates must be accessible at runtime (URI accessible)
   - Use modules in Bicep (cleaner than JSON linked templates)

6. **Bicep Advantages**
   - Cleaner, more readable syntax
   - Smaller file sizes
   - Recommended for new deployments
   - Automatically compiles to ARM JSON
   - Better tooling and validation

7. **Exporting Templates**
   - Export from existing resources useful for learning
   - Not guaranteed to be production-ready
   - Often need to add missing parameters
   - Hand-written templates better for production

8. **Validation**
   - Always validate before deployment
   - Use what-if to see changes
   - Test in non-production first
   - Check parameter values and API versions

### Exam Question Patterns

- **"Deploy multiple VMs consistently..."** → ARM template with parameters
- **"Reduce template duplication..."** → Linked templates or Bicep modules
- **"Preview changes before deployment..."** → Use what-if operation
- **"Export existing VM configuration..."** → Portal export or PowerShell Save-AzResourceGroupTemplate
- **"Deploy across environments..."** → Parameters file or template specs
- **"Simplify complex deployment..."** → Linked templates or Bicep

---

## Exam Scenarios

### Scenario 1: Deploy Multi-Tier Application Template

**Situation:**
Create a reusable template to deploy 3-tier application (web, app, database) with networking, load balancing, and scalability across different environments (dev/prod).

**Requirements:**
- Single main template coordinating multiple linked templates
- Parameters for customization (VM size, environment, location)
- Separate modules for each tier
- Outputs for connection strings and endpoints
- Support for both incremental and complete deployments

**Solution:**

**Main Template (main.json):**
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "environment": {
      "type": "string",
      "allowedValues": ["dev", "prod"],
      "defaultValue": "dev"
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]"
    },
    "vmSize": {
      "type": "string",
      "defaultValue": "[if(equals(parameters('environment'), 'prod'), 'Standard_E4s_v5', 'Standard_B2s')]"
    }
  },
  "variables": {
    "templateBaseUri": "https://raw.githubusercontent.com/myrepo/templates/",
    "deploymentName": "[concat('app-', parameters('environment'))]"
  },
  "resources": [
    {
      "type": "Microsoft.Resources/deployments",
      "apiVersion": "2020-10-01",
      "name": "networkingDeployment",
      "properties": {
        "mode": "Incremental",
        "templateLink": {
          "uri": "[concat(variables('templateBaseUri'), 'networking.json')]"
        },
        "parameters": {
          "environment": { "value": "[parameters('environment')]" },
          "location": { "value": "[parameters('location')]" }
        }
      }
    },
    {
      "type": "Microsoft.Resources/deployments",
      "apiVersion": "2020-10-01",
      "name": "webTierDeployment",
      "dependsOn": ["networkingDeployment"],
      "properties": {
        "mode": "Incremental",
        "templateLink": {
          "uri": "[concat(variables('templateBaseUri'), 'web-tier.json')]"
        },
        "parameters": {
          "vmSize": { "value": "[parameters('vmSize')]" },
          "environment": { "value": "[parameters('environment')]" },
          "vnetId": { "value": "[reference('networkingDeployment').outputs.vnetId.value]" }
        }
      }
    },
    {
      "type": "Microsoft.Resources/deployments",
      "apiVersion": "2020-10-01",
      "name": "appTierDeployment",
      "dependsOn": ["networkingDeployment"],
      "properties": {
        "mode": "Incremental",
        "templateLink": {
          "uri": "[concat(variables('templateBaseUri'), 'app-tier.json')]"
        },
        "parameters": {
          "vmSize": { "value": "[parameters('vmSize')]" },
          "environment": { "value": "[parameters('environment')]" },
          "vnetId": { "value": "[reference('networkingDeployment').outputs.vnetId.value]" }
        }
      }
    },
    {
      "type": "Microsoft.Resources/deployments",
      "apiVersion": "2020-10-01",
      "name": "databaseDeployment",
      "properties": {
        "mode": "Incremental",
        "templateLink": {
          "uri": "[concat(variables('templateBaseUri'), 'database.json')]"
        },
        "parameters": {
          "environment": { "value": "[parameters('environment')]" },
          "location": { "value": "[parameters('location')]" }
        }
      }
    }
  ],
  "outputs": {
    "webEndpoint": {
      "type": "string",
      "value": "[reference('webTierDeployment').outputs.loadBalancerIp.value]"
    },
    "databaseConnectionString": {
      "type": "string",
      "value": "[reference('databaseDeployment').outputs.connectionString.value]"
    },
    "environmentInfo": {
      "type": "object",
      "value": {
        "environment": "[parameters('environment')]",
        "location": "[parameters('location')]",
        "deploymentId": "[deployment().name]"
      }
    }
  }
}
```

**Deployment Command:**
```powershell
# Deploy to dev
New-AzResourceGroupDeployment `
    -Name "app-dev" `
    -ResourceGroupName "myResourceGroup" `
    -TemplateFile "main.json" `
    -environment "dev" `
    -location "eastus"

# Deploy to prod with larger VMs
New-AzResourceGroupDeployment `
    -Name "app-prod" `
    -ResourceGroupName "prodResourceGroup" `
    -TemplateFile "main.json" `
    -environment "prod" `
    -location "eastus" `
    -vmSize "Standard_E4s_v5"
```

---

### Scenario 2: Convert Bicep to ARM JSON for Compliance

**Situation:**
Organization requires all infrastructure deployments to use ARM JSON templates (for compliance/auditing). You have Bicep files but need to convert them for submission.

**Solution:**

```bash
# Decompile Bicep to ARM JSON
bicep build main.bicep --output-format json > azuredeploy.json

# Verify generated JSON
Test-AzResourceGroupDeployment -ResourceGroupName "rg" -TemplateFile "azuredeploy.json"

# Deploy generated JSON
New-AzResourceGroupDeployment -ResourceGroupName "rg" -TemplateFile "azuredeploy.json"
```

**Generated ARM JSON from Bicep will:**
- Expand all Bicep syntax to JSON
- Convert modules to nested templates
- Inline all variables and functions
- Maintain functionality and parameters

---

### Scenario 3: Export and Reuse Existing VM Template

**Situation:**
Your team manually created a complex VM in the portal. You need to export it to a template for reuse, but need to parameterize it for different environments.

**Solution:**

1. **Export from Portal:**
```powershell
Save-AzResourceGroupTemplate -ResourceGroupName "myResourceGroup" -Path "./"
```

2. **Enhance Exported Template:**
```json
// Add parameters for reusability
"parameters": {
  "vmName": {
    "type": "string",
    "defaultValue": "myVM"
  },
  "vmSize": {
    "type": "string",
    "defaultValue": "Standard_D2s_v3"
  },
  "adminUsername": {
    "type": "string"
  },
  "adminPassword": {
    "type": "securestring"
  }
}

// Replace hardcoded values with parameter references
"name": "[parameters('vmName')]",
"vmSize": "[parameters('vmSize')]",
"adminUsername": "[parameters('adminUsername')]",
"adminPassword": "[parameters('adminPassword')]"
```

3. **Create Parameters File:**
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "vmName": { "value": "myVM" },
    "vmSize": { "value": "Standard_D2s_v3" },
    "adminUsername": { "value": "azureuser" },
    "adminPassword": { "value": "P@ssw0rd123!" }
  }
}
```

4. **Deploy Parameterized Template:**
```powershell
New-AzResourceGroupDeployment `
    -ResourceGroupName "myResourceGroup" `
    -TemplateFile "azuredeploy.json" `
    -TemplateParameterFile "azuredeploy.parameters.json"
```

---

## Troubleshooting Guide

### Issue 1: Template Validation Error

**Symptoms:**
- InvalidTemplate error
- Property not recognized
- Missing required properties

**Common Causes:**
- Incorrect schema version
- Wrong API version for resource type
- Syntax errors in JSON
- Missing required properties

**Solutions:**

1. **Check Schema Version:**
```powershell
# Validate and see detailed errors
Test-AzResourceGroupDeployment `
    -ResourceGroupName "rg" `
    -TemplateFile "template.json" `
    -Verbose
```

2. **Verify Resource API Version:**
```powershell
# Find correct API version
Get-AzResourceProvider -ProviderNamespace "Microsoft.Compute" | `
    Select -ExpandProperty ResourceTypes | `
    Where-Object ResourceTypeName -eq "virtualMachines"
```

3. **Use VS Code Validation:**
- Install "Azure Resource Manager Tools" extension
- Real-time syntax checking and IntelliSense
- Validates against schema

---

### Issue 2: Parameter Not Recognized

**Symptoms:**
- "Parameter 'xxx' not found"
- Parameter values not applied
- Hardcoded values used instead

**Causes:**
- Parameter name mismatch
- Parameter not defined in template
- Case sensitivity in parameter names

**Solutions:**

```powershell
# Check parameter definition
(Get-Content "template.json" | ConvertFrom-Json).parameters.Keys

# Verify parameter names match exactly
New-AzResourceGroupDeployment `
    -ResourceGroupName "rg" `
    -TemplateFile "template.json" `
    -vmSize "Standard_D2s_v3"  # Must match parameter name exactly
```

---

### Issue 3: Linked Template Not Found

**Symptoms:**
- "Unable to retrieve linked template"
- "404 - Not Found"
- Network timeout during deployment

**Causes:**
- Template URI incorrect or unreachable
- Network/firewall blocking access
- Insufficient permissions to access storage account
- Template URI not accessible in target environment

**Solutions:**

1. **Verify Template URI Accessible:**
```powershell
# Test URL manually
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/myrepo/templates/storage.json"

# Use absolute URI with SAS token for storage accounts
$storageContext = New-AzStorageContext -StorageAccountName "mystg" -StorageAccountKey "key"
$tokenUri = New-AzStorageBlobSASToken -Container "templates" `
    -Blob "storage.json" -Context $storageContext -Permission r -ExpiryTime (Get-Date).AddDays(1)
```

2. **Use Relative Paths in Bicep:**
```bicep
module storage 'modules/storage.bicep' = {
  name: 'storageModule'
  // Bicep handles paths automatically
}
```

3. **Use Template Specs (Avoids URL Issues):**
```powershell
# Store template as resource (no URL needed)
New-AzTemplateSpec -ResourceGroupName "rg" `
    -Name "storage" -Version "1.0" `
    -TemplateFile "storage.json"

# Reference by resource ID (always accessible)
New-AzResourceGroupDeployment `
    -ResourceGroupName "app-rg" `
    -TemplateSpec "/subscriptions/xxx/resourceGroups/rg/providers/Microsoft.Resources/templateSpecs/storage/versions/1.0"
```

---

### Issue 4: Deployment Takes Too Long or Hangs

**Symptoms:**
- Deployment stuck in "Running" state
- No error messages
- Timeout after extended wait

**Causes:**
- Large deployment (100+ resources)
- Resource dependencies not optimized
- Circular dependencies
- Network connectivity issues

**Solutions:**

1. **Check Deployment Progress:**
```powershell
Get-AzResourceGroupDeployment -ResourceGroupName "rg" -Name "deployment" | 
    Select-Object -Property @{Name="State";Expression={$_.ProvisioningState}}, Duration
```

2. **Use What-If First:**
```powershell
# Preview without deploying
New-AzResourceGroupDeployment -ResourceGroupName "rg" -TemplateFile "template.json" -WhatIf
```

3. **Reduce Template Complexity:**
- Break into linked templates
- Deploy one resource type at a time if needed
- Check for circular dependencies

4. **Increase Timeout:**
```powershell
# PowerShell has built-in timeout management
New-AzResourceGroupDeployment `
    -ResourceGroupName "rg" `
    -TemplateFile "template.json" `
    -TimeoutInMinutes 120  # Extended timeout
```

---

## Related Topics

- **Configure VMs**: VM sizing, disks, encryption, availability
- **Create and Configure Containers**: Container Registry, ACI, Container Apps
- **Create and Configure Azure App Service**: App Service plans, deployment slots
- **Virtual Networking**: VNets, subnets, networking templates
- **Implement Backup and Recovery**: Backup templates and vault configuration

---

## Key References

- [What are ARM templates?](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/overview)
- [Understand ARM template structure](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/syntax)
- [Bicep documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview)
- [Linked and nested templates](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/linked-templates)
- [Template reference](https://learn.microsoft.com/en-us/azure/templates/)
- [Azure Resource Manager GitHub](https://github.com/Azure/azure-quickstart-templates)
