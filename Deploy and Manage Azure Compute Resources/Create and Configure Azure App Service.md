# Create and Configure Azure App Service

## Table of Contents
1. [App Service Overview](#app-service-overview)
2. [App Service Plans](#app-service-plans)
3. [Creating App Service Applications](#creating-app-service-applications)
4. [Scaling App Service](#scaling-app-service)
5. [Deployment Slots](#deployment-slots)
6. [Custom Domains and TLS/SSL](#custom-domains-and-tlsssl)
7. [Backup and Recovery](#backup-and-recovery)
8. [Authentication and Authorization](#authentication-and-authorization)
9. [PowerShell & CLI Commands](#powershell--cli-commands)
10. [Exam Tips](#exam-tips)
11. [Exam Scenarios](#exam-scenarios)
12. [Troubleshooting](#troubleshooting)

---

## App Service Overview

Azure App Service is a fully managed, scalable web hosting service for building web apps, mobile back ends, and RESTful APIs. Key characteristics:

### Benefits
- **Platform as a Service (PaaS)**: Manage application, not infrastructure
- **Multi-language support**: .NET, Java, Node.js, Python, PHP, Ruby
- **Multi-platform**: Deploy to Windows or Linux containers
- **Built-in capabilities**: Authentication, autoscaling, CI/CD, monitoring
- **Global infrastructure**: Deploy to 200+ regions worldwide
- **High availability**: SLA of 99.95% in Standard tier and higher

### Supported Application Types
- Web apps (ASP.NET, Java, Node.js, Python, PHP, Ruby)
- Mobile back ends (iOS, Android, Windows)
- REST APIs
- Static websites
- Web App for Containers (Docker)
- Azure Functions (serverless compute)

---

## App Service Plans

An App Service plan defines the compute resources available to your apps.

### Pricing Tiers

| Category | Tiers | Description |
|----------|-------|-------------|
| **Shared Compute** | Free, Shared | Shared resources with other customers' apps; CPU quotas allocated per app; no scale-out capability; development/testing only |
| **Dedicated Compute** | Basic, Standard, Premium, PremiumV2, PremiumV3, PremiumV4 | Dedicated VMs for your apps; run on dedicated hardware; can scale out (more instances); production workloads |
| **Isolated** | IsolatedV2 | Dedicated VMs on dedicated VNets; maximum isolation; 100+ instances possible; enterprise workloads |

### Tier Feature Comparison

| Feature | Basic | Standard | Premium | PremiumV2 | PremiumV3 | PremiumV4 | Isolated |
|---------|-------|----------|---------|-----------|-----------|-----------|----------|
| Max Scale Out | 3 | 10 | 30 | 30 | 30 | 30 | 100 |
| Deployment Slots | — | 5 | 20 | 20 | 20 | 20 | 20 |
| Custom Domains | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Managed Certificates | — | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Auto Scale | — | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Always On | — | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Automatic Scaling | — | — | — | ✓ | ✓ | ✓ | — |

### Scale-Up vs Scale-Out

**Scale-Up (Vertical Scaling)**
- Change App Service plan tier (more CPU, memory, disk)
- Affects all apps in the plan
- Takes only seconds to apply
- Change pricing tier in Azure portal under "Scale up"

**Scale-Out (Horizontal Scaling)**
- Increase number of VM instances running your app
- Manual scaling: Change instance count in App Service plan
- Automatic scaling: Configure rules based on metrics
- Platform-managed autoscale for Premium V2, V3, V4 tiers
- Availability Sets/Zones distribute instances

### Creating App Service Plans

**Using Azure Portal**
1. Create resource → App Service Plan
2. Select subscription, resource group, region
3. Name the plan (e.g., myappserviceplan)
4. Select OS (Windows or Linux)
5. Choose pricing tier and instance count
6. Click Create

**Using PowerShell**
```powershell
# Create App Service plan
New-AzAppServicePlan -ResourceGroupName myResourceGroup `
  -Name myAppServicePlan `
  -Location eastus `
  -Tier Standard `
  -WorkerSize Small `
  -NumberofWorkers 1

# Create plan with per-app scaling
New-AzAppServicePlan -ResourceGroupName myResourceGroup `
  -Name myAppServicePlan `
  -Location eastus `
  -Tier Premium `
  -WorkerSize Small `
  -NumberofWorkers 5 `
  -PerSiteScaling $true
```

**Using Azure CLI**
```azurecli
# Create App Service plan
az appservice plan create \
  --name myAppServicePlan \
  --resource-group myResourceGroup \
  --sku B1 \
  --is-linux

# Create Windows plan
az appservice plan create \
  --name myAppServicePlan \
  --resource-group myResourceGroup \
  --sku S1
```

---

## Creating App Service Applications

### Web App Creation

**Using Azure Portal**
1. Create resource → App Service or Web App
2. Select subscription, resource group
3. Name your web app (must be globally unique)
4. Select publish method (Code or Docker Container)
5. Select runtime (Node, Python, Java, .NET, PHP, Ruby)
6. Select App Service plan
7. Configure monitoring (Application Insights optional)
8. Review + Create

**Using PowerShell**
```powershell
# Create web app
New-AzWebApp -ResourceGroupName myResourceGroup `
  -Name myWebApp `
  -Location eastus `
  -AppServicePlan myAppServicePlan

# Create web app with container
New-AzWebApp -ResourceGroupName myResourceGroup `
  -Name myWebApp `
  -Location eastus `
  -AppServicePlan myAppServicePlan `
  -ContainerImageName myregistry.azurecr.io/myapp:latest `
  -ContainerRegistryUrl https://myregistry.azurecr.io `
  -ContainerRegistryUser username `
  -ContainerRegistryPassword password
```

**Using Azure CLI**
```azurecli
# Create web app
az webapp create \
  --resource-group myResourceGroup \
  --plan myAppServicePlan \
  --name myWebApp \
  --runtime "node|18-lts"

# Create with container image
az webapp create \
  --resource-group myResourceGroup \
  --plan myAppServicePlan \
  --name myWebApp \
  --deployment-container-image-name myregistry.azurecr.io/myapp:latest
```

### App Settings and Configuration

Configure application-specific settings without modifying code.

**Using Portal**
1. Select app → Settings → Configuration
2. Add new application setting or connection string
3. Select stack settings for language runtime
4. Configure handler mappings

**Using PowerShell**
```powershell
# Add app setting
$webapp = Get-AzWebApp -ResourceGroupName myResourceGroup -Name myWebApp
$webapp.SiteConfig.AppSettings.Add(@{ Name = "MyKey"; Value = "MyValue" })
Set-AzWebApp -WebApp $webapp

# Add connection string
$webapp = Get-AzWebApp -ResourceGroupName myResourceGroup -Name myWebApp
$webapp.SiteConfig.ConnectionStrings.Add(@{ 
    Name = "MyConnection"
    ConnectionString = "Data Source=myserver.database.windows.net;..."
    Type = "SQLAzure"
})
Set-AzWebApp -WebApp $webapp
```

**Using Azure CLI**
```azurecli
# Add app setting
az webapp config appsettings set \
  --resource-group myResourceGroup \
  --name myWebApp \
  --settings MyKey=MyValue

# Add connection string
az webapp config connection-string set \
  --resource-group myResourceGroup \
  --name myWebApp \
  --connection-string-type SQLAzure \
  --settings MyConnection="connection-string-value"
```

---

## Scaling App Service

### Manual Scaling

**Scale-Up (Change Pricing Tier)**
1. App Service → Scale up
2. Select desired tier
3. Click Select to apply (seconds to apply)

**Scale-Out (Add Instances)**
1. App Service → Scale out
2. Manually increase instance count slider
3. Click Save

**Using PowerShell**
```powershell
# Scale-up to Premium tier
Set-AzAppServicePlan -ResourceGroupName myResourceGroup `
  -Name myAppServicePlan `
  -Tier Premium

# Scale-out: add instances
$appServicePlan = Get-AzAppServicePlan -ResourceGroupName myResourceGroup `
  -Name myAppServicePlan
$appServicePlan.NumberofWorkers = 5
Set-AzAppServicePlan -AppServicePlan $appServicePlan
```

### Automatic Scaling

Available in Standard tier and higher. Uses Azure Autoscale with metric-based rules.

**Scale Rules**
- CPU percentage > 80% → scale out
- Memory usage > 90% → scale out
- Requests > 100 → scale out
- Low traffic → scale in (cooldown 5 minutes)

**Using Portal**
1. App Service → Scale out (App Service Plan)
2. Click "Enable autoscale"
3. Create scale rule based on metric
4. Define minimum/maximum instance counts
5. Set cooldown period (default 5 minutes)

**Using PowerShell**
```powershell
# Enable per-app scaling for Premium plan
Set-AzAppServicePlan -ResourceGroupName myResourceGroup `
  -Name myAppServicePlan `
  -PerSiteScaling $true

# Configure autoscale rule (via Azure Monitor)
$rule = New-AzAutoscaleRule `
  -MetricName "CpuPercentage" `
  -MetricResourceId "/subscriptions/sub-id/resourceGroups/myResourceGroup/providers/Microsoft.Web/serverfarms/myAppServicePlan" `
  -Operator GreaterThan `
  -MetricStatistic Average `
  -Threshold 80 `
  -TimeGrain 00:01:00 `
  -TimeWindow 00:05:00 `
  -ScaleActionCooldown 00:05:00 `
  -ScaleActionDirection Increase `
  -ScaleActionValue 1
```

### Automatic Scale-Out (Platform Managed)

**Available in Premium V2, V3, V4 tiers only**
- Platform automatically manages scaling based on HTTP traffic
- Prewarms instances before traffic spike
- Smooth scaling without cold starts
- Billed per second for all instances including prewarmed
- Default minimum replicas: 1; prewarmed instances: 1

**Configuration**
1. App Service → Scale out
2. Select Automatic in dropdown
3. Configure minimum/maximum instances
4. Set prewarmed instance count
5. Click Save

---

## Deployment Slots

Staging environments for testing before production deployment.

### Slot Capabilities

| Feature | Availability |
|---------|--------------|
| Number of slots | Depends on tier (5 for Standard, 20 for Premium) |
| Auto swap | Standard+ |
| Swap with preview | Standard+ |
| Separate configuration | Yes |
| Custom domains | Sticky to slot |
| TLS/SSL certificates | Not swapped |
| Scale settings | Not swapped |

### Creating Deployment Slots

**Using Portal**
1. App Service → Deployment slots
2. Click "Add Slot"
3. Name slot (e.g., staging, test)
4. Choose to clone settings from production
5. Click Add

**Using PowerShell**
```powershell
# Create deployment slot
New-AzWebAppSlot -ResourceGroupName myResourceGroup `
  -Name myWebApp `
  -Slot staging

# Deploy to slot
Publish-AzWebapp -ResourceGroupName myResourceGroup `
  -Name myWebApp `
  -Slot staging `
  -ArchivePath myapp.zip
```

**Using Azure CLI**
```azurecli
# Create deployment slot
az webapp deployment slot create \
  --resource-group myResourceGroup \
  --name myWebApp \
  --slot staging

# Deploy to slot
az webapp deployment source config-zip \
  --resource-group myResourceGroup \
  --name myWebApp \
  --slot staging \
  --src-url https://myapp.zip
```

### Slot Swapping

**Swap Operation Steps**
1. Apply target slot settings to source slot (slot-specific settings like app settings, connection strings)
2. Restart all instances in source slot
3. Trigger application initialization (warmup)
4. Swap routing (source becomes target)
5. Apply source slot settings to target slot

**Swapped Settings** (moved between slots)
- Language framework versions (.NET, Java, Node.js, PHP, Python)
- WebSockets enabled/disabled
- App settings
- Connection strings
- Mounted storage accounts
- Handler mappings
- Public certificates
- WebJobs content
- Hybrid connections
- Virtual network integration

**Sticky Settings** (NOT swapped)
- Protocol settings (HTTPS Only, TLS version)
- Custom domain names
- Certificates (nonpublic)
- Scale settings
- Always On
- IP restrictions
- Diagnostic settings

**Performing a Swap**

Using Portal:
1. Deployment slots → Select slot → Swap
2. Choose target slot (usually production)
3. Review settings to be swapped
4. Click "Start Swap"

**Using PowerShell**
```powershell
# Swap slots
Switch-AzWebAppSlot -ResourceGroupName myResourceGroup `
  -Name myWebApp `
  -SourceSlotName staging `
  -DestinationSlotName production
```

**Using Azure CLI**
```azurecli
# Swap slots
az webapp deployment slot swap \
  --resource-group myResourceGroup \
  --name myWebApp \
  --slot staging \
  --target-slot production
```

### Swap with Preview

Validate changes before completing swap.

**Process**
1. Apply target settings to source slot
2. Warm up source slot instances
3. Pause swap (allow validation)
4. Complete swap or cancel

**Using Azure CLI**
```azurecli
# Start preview swap
az webapp deployment slot swap \
  --resource-group myResourceGroup \
  --name myWebApp \
  --slot staging \
  --target-slot production \
  --action preview

# Complete swap
az webapp deployment slot swap \
  --resource-group myResourceGroup \
  --name myWebApp \
  --slot staging \
  --target-slot production \
  --action swap

# Cancel swap
az webapp deployment slot swap \
  --resource-group myResourceGroup \
  --name myWebApp \
  --slot staging \
  --target-slot production \
  --action reset
```

### Auto Swap

Automatically swap to production after successful deployment.

**Using Portal**
1. Deployment slot → Settings → Configuration → General settings
2. Set "Auto swap enabled" to On
3. Select target slot
4. Save changes

**Using PowerShell**
```powershell
Set-AzWebAppSlot -ResourceGroupName myResourceGroup `
  -Name myWebApp `
  -Slot staging `
  -AutoSwapSlotName production
```

---

## Custom Domains and TLS/SSL

### Adding Custom Domains

**Prerequisites**
- Own or control the domain
- Domain registration with DNS provider
- DNS records configured

**Using Portal**
1. App Service → Custom domains
2. Click "Add custom domain"
3. Enter fully qualified domain name (e.g., www.contoso.com)
4. Select validation method:
   - A record: Point IP to app
   - CNAME: Point domain to azurewebsites.net
5. Create DNS records in your domain provider
6. Click Validate
7. Add binding

**Configuration for Different Domain Types**

Apex Domain (contoso.com):
- Use A record pointing to app's IP address
- Create TXT record for domain verification

Subdomain (www.contoso.com):
- Use CNAME record pointing to `<app-name>.azurewebsites.net`
- Must point directly to azurewebsites.net (no intermediate CNAME)

Wildcard Domain (*.contoso.com):
- Use CNAME record like subdomain
- Matches any subdomain automatically

### TLS/SSL Certificates

**Certificate Types**

| Type | Cost | Management | Use Case |
|------|------|-----------|----------|
| **App Service Managed** | Free | Automatic renewal | Simplest, no manual management |
| **App Service Certificate** | Paid | You manage | Full control, exportable |
| **Bring Your Own (BYOC)** | Your cost | You manage | Use existing certificates |
| **Key Vault Imported** | Your cost | Key Vault manages | Centralized secret management |

**Creating Free Managed Certificate**

Prerequisites:
- Basic tier or higher
- Public domain on app
- No app authentication enabled
- Custom domain configured with A record or direct CNAME

**Using Portal**
1. Custom domains → Select domain → Add binding
2. For TLS/SSL certificate, select "Create App Service Managed Certificate"
3. Click Validate and Add
4. Certificate issued and renewed automatically

**Using CLI**
```azurecli
# Create managed certificate for custom domain
az webapp config ssl create \
  --resource-group myResourceGroup \
  --name myWebApp \
  --certificate-name mycertificate \
  --certificate-thumbprint cert-thumbprint
```

### TLS/SSL Bindings

Two binding types:

**SNI SSL (Server Name Indication)**
- Multiple certificates on one IP address
- Modern browsers support SNI
- Recommended approach
- Works with public/private certificates

**IP SSL (IP-based SSL)**
- One certificate per IP address
- Dedicated IP address created
- Standard tier and higher only
- Update A record to new IP

**Using Portal**
1. Custom domains → Select domain → Add binding
2. Choose SNI SSL or IP SSL
3. Select or upload certificate
4. Click Add

**Using PowerShell**
```powershell
# Get certificate and create binding
$cert = Get-AzWebAppCertificate -ResourceGroupName myResourceGroup `
  -Thumbprint "cert-thumbprint"

New-AzWebAppSSLBinding -ResourceGroupName myResourceGroup `
  -WebAppName myWebApp `
  -Thumbprint "cert-thumbprint" `
  -Name "www.contoso.com" `
  -SslState SniEnabled

# Remove binding
Remove-AzWebAppSSLBinding -ResourceGroupName myResourceGroup `
  -WebAppName myWebApp `
  -Name "www.contoso.com"
```

---

## Backup and Recovery

### Backup Configuration

**Backup Requirements**
- Standard tier and higher (not Free/Shared)
- Associated Azure Storage account
- Can store up to 50 backups (by plan)
- Database backups included (SQL, MySQL)

**Backup Scope**
- App configuration
- File content (code, static assets)
- Databases (SQL Database, MySQL)
- NOT included: TLS certificates, diagnostic settings

**Using Portal**
1. App Service → Backup
2. Click Configure
3. Select storage account and container
4. Choose backup frequency (hourly, daily, weekly)
5. Set retention period (30 days default)
6. Click Save

**Using PowerShell**
```powershell
# Create backup
$context = New-AzStorageContext -StorageAccountName mystorageaccount `
  -StorageAccountKey $storageKey

Backup-AzWebApp -ResourceGroupName myResourceGroup `
  -Name myWebApp `
  -StorageContext $context `
  -ContainerName backups

# List backups
$context = New-AzStorageContext -StorageAccountName mystorageaccount `
  -StorageAccountKey $storageKey

Get-AzWebAppBackup -ResourceGroupName myResourceGroup `
  -Name myWebApp
```

### Restoring from Backup

**Using Portal**
1. App Service → Backup
2. Click "Restore a backup"
3. Select backup date/time
4. Choose to overwrite current app or restore to new app
5. Click Restore

**Using PowerShell**
```powershell
# Get backup
$backup = Get-AzWebAppBackup -ResourceGroupName myResourceGroup `
  -Name myWebApp | Select -First 1

# Restore backup to same app
Restore-AzWebApp -ResourceGroupName myResourceGroup `
  -Name myWebApp `
  -Backup $backup `
  -Overwrite

# Restore to new app
Restore-AzWebApp -ResourceGroupName myResourceGroup `
  -Name myWebApp `
  -Backup $backup `
  -TargetAppServicePlan newAppServicePlan
```

### Disaster Recovery Planning

**Multi-Region Deployment**
1. Deploy app to secondary region
2. Configure traffic routing (Traffic Manager or Front Door)
3. Regular backup replication to secondary region
4. DNS failover mechanism

**Backup Retention**
- Set retention period based on RPO (Recovery Point Objective)
- Daily backups = 1-day RPO
- Test backup restoration regularly

---

## Authentication and Authorization

### Built-in Authentication (Easy Auth)

Azure App Service provides platform-level authentication without writing code.

**Supported Providers**
- Microsoft Entra ID (Azure AD)
- Microsoft accounts
- Facebook
- Google
- Twitter (X)

**Benefits**
- No code changes required
- Centralized identity management
- Token validation and refresh
- Multi-provider support
- Secure credential storage

### Enabling Authentication

**Using Portal**
1. App Service → Settings → Authentication
2. Click "Add identity provider"
3. Select provider (Entra ID recommended)
4. Configure app registration
5. Set unauthenticated request behavior

**Using PowerShell**
```powershell
# Enable Entra ID authentication
$authConfig = @{
    enabled = $true
    defaultProvider = "AzureActiveDirectory"
    identityProviders = @{
        azureActiveDirectory = @{
            enabled = $true
            registration = @{
                openIdIssuer = "https://login.microsoftonline.com/{tenantId}/v2.0"
                clientId = "client-id"
                clientSecretSettingName = "MICROSOFT_PROVIDER_AUTHENTICATION_SECRET"
            }
        }
    }
    login = @{
        logout = "/logout"
    }
}

Update-AzWebApp -ResourceGroupName myResourceGroup `
  -Name myWebApp `
  -HttpLoggingEnabled $true
```

### Access Control

**Request Handling Options**

| Action | Behavior |
|--------|----------|
| Allow anonymous requests | Passes unauthenticated requests to app code |
| Require authentication | Redirects unauthenticated users to login |
| HTTP 401 Unauthorized | Returns 401 for API requests |
| HTTP 403 Forbidden | Returns 403 for all unauthenticated requests |

**Using Portal**
1. Authentication → Unauthenticated requests action
2. Choose redirect to login provider or allow anonymous

### Managed Identity

Use Azure-managed identity for app-to-service authentication (no credentials).

**System-Assigned Managed Identity**
- Automatically created with app
- Automatically deleted when app deleted
- One per app

**User-Assigned Managed Identity**
- Created separately
- Shared across resources
- Lifecycle managed independently

**Using Portal**
1. App Service → Identity
2. System assigned: Toggle On
3. User assigned: Click "Add user-assigned managed identity"

**Using PowerShell**
```powershell
# Enable system-assigned managed identity
Set-AzWebApp -ResourceGroupName myResourceGroup `
  -Name myWebApp `
  -AssignIdentity $true

# Get managed identity
$webApp = Get-AzWebApp -ResourceGroupName myResourceGroup `
  -Name myWebApp
$webApp.Identity.PrincipalId  # Use this for RBAC roles
```

---

## PowerShell & CLI Commands

### Creating and Configuring Apps

```powershell
# Create App Service plan
New-AzAppServicePlan -ResourceGroupName myRG -Name myPlan `
  -Location eastus -Tier Standard -NumberofWorkers 1

# Create web app
New-AzWebApp -ResourceGroupName myRG -Name myApp `
  -AppServicePlan myPlan -Location eastus

# Set app settings
$webapp = Get-AzWebApp -ResourceGroupName myRG -Name myApp
$webapp.SiteConfig.AppSettings.Add(@{ Name = "key"; Value = "value" })
Set-AzWebApp -WebApp $webapp

# Get app configuration
Get-AzWebApp -ResourceGroupName myRG -Name myApp | 
  Select-Object -ExpandProperty SiteConfig
```

### Scaling Commands

```powershell
# Scale-up to Premium tier
Set-AzAppServicePlan -ResourceGroupName myRG -Name myPlan -Tier Premium

# Scale-out: add instances
$plan = Get-AzAppServicePlan -ResourceGroupName myRG -Name myPlan
$plan.NumberofWorkers = 5
Set-AzAppServicePlan -AppServicePlan $plan

# Enable per-app scaling
Set-AzAppServicePlan -ResourceGroupName myRG -Name myPlan `
  -PerSiteScaling $true
```

### Deployment Slots

```powershell
# Create slot
New-AzWebAppSlot -ResourceGroupName myRG -Name myApp -Slot staging

# Swap slots
Switch-AzWebAppSlot -ResourceGroupName myRG -Name myApp `
  -SourceSlotName staging -DestinationSlotName production

# Auto swap
Set-AzWebAppSlot -ResourceGroupName myRG -Name myApp -Slot staging `
  -AutoSwapSlotName production
```

### Backup and Restore

```powershell
# Create backup
$storageContext = New-AzStorageContext -StorageAccountName myStorage `
  -StorageAccountKey $key
Backup-AzWebApp -ResourceGroupName myRG -Name myApp `
  -StorageContext $storageContext -ContainerName backups

# Restore backup
$backup = Get-AzWebAppBackup -ResourceGroupName myRG -Name myApp |
  Select -First 1
Restore-AzWebApp -ResourceGroupName myRG -Name myApp -Backup $backup `
  -Overwrite
```

### Azure CLI Commands

```bash
# Create App Service plan
az appservice plan create --name myPlan --resource-group myRG \
  --sku S1 --location eastus

# Create web app
az webapp create --resource-group myRG --plan myPlan --name myApp

# Add app setting
az webapp config appsettings set --resource-group myRG --name myApp \
  --settings myKey=myValue

# Scale-out
az appservice plan update --resource-group myRG --name myPlan \
  --number-of-workers 5

# Create deployment slot
az webapp deployment slot create --resource-group myRG --name myApp \
  --slot staging

# Swap slots
az webapp deployment slot swap --resource-group myRG --name myApp \
  --slot staging --target-slot production
```

---

## Exam Tips

### Key Concepts
1. **Pricing Tiers**: Free/Shared for dev; Standard+ for production; Premium V3/V4 for autoscaling
2. **Scaling Strategy**: Scale-up changes tier; scale-out adds instances; autoscale uses metrics
3. **Deployment Slots**: Use for zero-downtime deployments; swap settings but sticky settings remain
4. **Always On**: Required for continuous operation; only available in Basic+ tiers
5. **Managed Identity**: Preferred over connection strings for Azure service authentication
6. **TLS/SSL**: Free managed certificates for ease; bring your own for complex scenarios

### Common Scenarios
- **Cost optimization**: Use shared tiers for dev; Standard for production; Reserved instances for long-term
- **High availability**: Multi-region deployment with Traffic Manager; deployment slots for zero-downtime
- **Zero-downtime deployments**: Use deployment slots with swap; auto swap for CI/CD automation
- **Authentication**: Use built-in Entra ID authentication without custom code
- **Scaling**: Use autoscale rules for variable load; automatic scaling (Premium V3) for HTTP traffic spikes

### Important Limits
- Standard tier: 5 deployment slots
- Premium tier: 20 deployment slots
- Basic tier: 3 instances max scale-out
- Standard tier: 10 instances max scale-out
- Premium tier: 30 instances max scale-out
- Isolated tier: 100+ instances possible

---

## Exam Scenarios

### Scenario 1: E-Commerce Web Application Deployment

**Requirements**
- Deploy ASP.NET web app to Azure App Service
- Zero-downtime deployments for updates
- Automatically scale based on traffic
- Custom domain with HTTPS
- Backup and disaster recovery

**Solution**

1. **Create App Service Plan**
   - Tier: Standard or Premium (autoscaling needed)
   - Redundancy: Deploy to multiple regions
   - ```powershell
     New-AzAppServicePlan -ResourceGroupName ecommerce-rg `
       -Name ecommerce-plan -Location eastus -Tier Standard `
       -NumberofWorkers 2
     ```

2. **Create Web App**
   - ```powershell
     New-AzWebApp -ResourceGroupName ecommerce-rg -Name ecommerce-app `
       -AppServicePlan ecommerce-plan
     ```

3. **Configure Deployment Slots**
   - Create staging slot for testing
   - ```powershell
     New-AzWebAppSlot -ResourceGroupName ecommerce-rg `
       -Name ecommerce-app -Slot staging
     ```

4. **Setup Automatic Scaling**
   - CPU > 70% → scale out
   - Requests > 100 → scale out
   - Scale-in cooldown: 5 minutes

5. **Configure Custom Domain & HTTPS**
   - Add custom domain (e-commerce.com)
   - Create free managed certificate
   - Set HTTPS-only mode

6. **Setup Backup**
   - Daily backups to Azure Storage
   - Retention: 30 days
   - Test restoration monthly

7. **Deployment Process**
   - Deploy to staging slot
   - Run smoke tests
   - Swap staging → production (auto-swap option)

**Key Points**: Use deployment slots for zero-downtime; autoscale for variable load; daily backups for DR

---

### Scenario 2: Mobile Back-End API Service

**Requirements**
- RESTful API for mobile apps
- Multiple environments (dev, staging, production)
- Automatic scaling for variable traffic
- Managed identity for Azure SQL Database access
- Monitoring and alerting

**Solution**

1. **Create Premium Plan with Autoscale**
   - Tier: Premium V2 (autoscaling support)
   - Minimum instances: 2
   - Maximum instances: 10

2. **Create Web API App**
   - Runtime: .NET 8
   - Configure HTTPS-only

3. **Setup Deployment Slots**
   - Development slot
   - Staging slot
   - Production slot

4. **Configure Managed Identity**
   - Enable system-assigned identity
   - Grant SQL Database Contributor role
   ```powershell
   Set-AzWebApp -ResourceGroupName mobile-rg -Name mobile-api `
     -AssignIdentity $true
   ```

5. **Configure Autoscaling**
   - Metric: CPU > 75% → scale out (+1 instance)
   - Metric: CPU < 25% → scale in (-1 instance)
   - Cooldown: 5 minutes

6. **Application Settings**
   - Database connection string (use managed identity token)
   - API keys for external services
   - Feature flags

7. **Monitoring**
   - Application Insights integration
   - Set up alerts for error rate > 5%
   - Track response time SLA

**Key Points**: Managed identity for secure DB access; autoscaling for cost optimization; multiple slots for progressive deployment

---

### Scenario 3: Static Website with CDN

**Requirements**
- Host static website (HTML, CSS, JavaScript)
- Global distribution with CDN
- Custom domain with SSL
- Cost-effective solution
- Continuous deployment from GitHub

**Solution**

1. **Create App Service Plan**
   - Tier: Standard (minimal cost)
   - Single instance sufficient

2. **Create Web App**
   ```powershell
   New-AzWebApp -ResourceGroupName static-rg -Name static-site `
     -AppServicePlan static-plan
   ```

3. **Enable Static Website Hosting**
   - Deploy code from GitHub (CI/CD)
   - Or use deployment slots for staged releases

4. **Add Custom Domain**
   - Add domain (example.com)
   - Create free managed certificate

5. **Configure Azure CDN**
   - Add CDN endpoint
   - Origin: static-site.azurewebsites.net
   - Enable HTTPS
   - Set caching rules (aggressive for static content)

6. **Setup Continuous Deployment**
   - GitHub integration
   - Auto-deploy on main branch push
   - Or use deployment slots with swap

7. **Monitoring**
   - CDN performance metrics
   - Origin health monitoring
   - Cache hit ratio tracking

**Cost Optimization**: Use Shared tier ($9/month) for static content; CDN reduces origin bandwidth costs

**Key Points**: CDN for global distribution; CI/CD for automation; minimal compute needs for static content

---

## Troubleshooting

### Common Issues

**Issue: App doesn't start / 502 Bad Gateway**

*Causes*
- Insufficient memory/CPU
- Port mismatch
- Runtime not installed
- Dependency missing

*Solutions*
```powershell
# Check app settings
$app = Get-AzWebApp -ResourceGroupName myRG -Name myApp
$app.SiteConfig

# Check if Always On is enabled
$app.SiteConfig.AlwaysOn

# Scale up to higher tier if resource constrained
Set-AzAppServicePlan -ResourceGroupName myRG -Name myPlan `
  -Tier Premium
```

---

**Issue: Slow performance / High response times**

*Causes*
- Insufficient instances
- Database slow queries
- Memory leaks in application
- Network latency

*Solutions*
1. Scale out: increase instance count
2. Enable autoscaling for dynamic load
3. Use Application Insights to profile code
4. Add caching (Redis)
5. Database optimization and indexing
6. CDN for static content

---

**Issue: Deployment failures**

*Causes*
- Authentication credentials incorrect
- Incompatible runtime version
- Missing dependencies
- Large deployment package

*Solutions*
```powershell
# Verify deployment source
Get-AzWebAppPublishingProfile -ResourceGroupName myRG `
  -Name myApp -OutputFile profile.xml

# Check deployment logs
Get-AzWebAppDeployment -ResourceGroupName myRG -Name myApp `
  -Slot production | Select -First 5

# Clear deployment cache
Remove-AzWebApp -ResourceGroupName myRG -Name myApp
```

---

**Issue: Custom domain not working / DNS errors**

*Causes*
- DNS records not properly configured
- CNAME pointing to wrong target
- Validation record missing
- Domain not verified

*Solutions*
1. Verify DNS records:
   - A record: should point to app's IP
   - CNAME: should point to azurewebsites.net
   - TXT: domain verification record
2. Wait for DNS propagation (15-60 minutes)
3. Clear browser cache and DNS cache
4. Use nslookup to verify DNS resolution

---

**Issue: Certificate issues / SSL errors**

*Causes*
- Certificate expired
- Certificate not bound to domain
- Mixed HTTP/HTTPS content
- Client certificate requirement

*Solutions*
```powershell
# Check certificate status
Get-AzWebAppCertificate -ResourceGroupName myRG `
  -Thumbprint "cert-thumbprint"

# Renew managed certificate (automatic)
# Check renewal status in portal

# Create new binding if missing
New-AzWebAppSSLBinding -ResourceGroupName myRG -WebAppName myApp `
  -Thumbprint "cert-thumbprint" -Name "www.example.com" `
  -SslState SniEnabled
```

---

**Issue: Authentication not working**

*Causes*
- Authentication not enabled
- App registration not configured
- Redirect URI mismatch
- Token expired

*Solutions*
1. Verify authentication is enabled: Settings → Authentication
2. Check app registration in Entra ID
3. Verify redirect URI matches app domain
4. Clear cookies and try again
5. Check authentication logs in Application Insights

---

### Performance Optimization

**Strategies**
1. **Caching**: Redis for session/output caching
2. **CDN**: Cloudflare or Azure CDN for static assets
3. **Compression**: Enable gzip compression
4. **Async operations**: Use async/await for I/O operations
5. **Connection pooling**: Database connection reuse
6. **Profiling**: Application Insights for bottleneck identification
7. **Image optimization**: Resize/compress before upload
8. **Minification**: Minimize JavaScript, CSS files

---

**Monitoring Commands**

```powershell
# Get app metrics
Get-AzWebAppMetrics -ResourceGroupName myRG -Name myApp

# Get app health
Get-AzWebApp -ResourceGroupName myRG -Name myApp |
  Select-Object State, RepositorySiteName, DefaultHostName

# View Activity Log
Get-AzActivityLog -ResourceGroupName myRG -WarningAction SilentlyContinue
```

---

**Performance Tuning Checklist**
- [ ] App Service plan tier appropriate for load
- [ ] Autoscaling configured for variable traffic
- [ ] Always On enabled for production
- [ ] Application Insights monitoring enabled
- [ ] Caching implemented (output, query)
- [ ] CDN configured for static content
- [ ] Connection pooling enabled
- [ ] Deployment slots for zero-downtime releases
- [ ] Regular backups enabled and tested
- [ ] Log retention policy configured

