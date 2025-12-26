# Create and Configure Containers - Azure Administrator Exam Prep (AZ-104)

## Table of Contents
1. [Containers Overview](#containers-overview)
2. [Azure Container Registry](#azure-container-registry)
3. [Azure Container Instances](#azure-container-instances)
4. [Azure Container Apps](#azure-container-apps)
5. [Container Registry Operations](#container-registry-operations)
6. [Container Deployment](#container-deployment)
7. [Container Networking](#container-networking)
8. [Container Scaling](#container-scaling)
9. [PowerShell & CLI Commands](#powershell--cli-commands)
10. [Exam Tips & Key Concepts](#exam-tips--key-concepts)
11. [Exam Scenarios](#exam-scenarios)
12. [Troubleshooting Guide](#troubleshooting-guide)

---

## Containers Overview

### What Are Containers?

Containers are lightweight, portable computing environments that bundle application code, runtime, system tools, libraries, and settings into a single package.

**Container Benefits:**
- **Consistency**: Same behavior across development, testing, and production
- **Portability**: Run on any machine with container runtime (Docker, containerd)
- **Efficiency**: Lightweight compared to VMs, share host OS kernel
- **Speed**: Start in seconds vs minutes for VMs
- **Isolation**: Containers isolated from each other
- **Scalability**: Easy horizontal scaling by adding container instances
- **Microservices**: Perfect for distributed application architectures

### Container vs Virtual Machine

| Aspect | Container | Virtual Machine |
|--------|-----------|-----------------|
| **Size** | Megabytes | Gigabytes |
| **Startup Time** | Seconds | Minutes |
| **Resource Usage** | Minimal | Significant |
| **OS Isolation** | Process-level | Full OS |
| **Density** | High (100s) | Low (10s) |
| **Management** | Simple | Complex |
| **Persistence** | Stateless preferred | Stateful |

### Azure Container Services

Azure offers three main container hosting services:

1. **Azure Container Registry (ACR)**: Private container image repository
2. **Azure Container Instances (ACI)**: Serverless containers without orchestration
3. **Azure Container Apps (ACA)**: Container apps with built-in orchestration and scaling

---

## Azure Container Registry

### Registry Overview

Azure Container Registry (ACR) is a private Docker registry for storing and managing container images.

**Key Features:**
- Private image repository
- Image versioning and tagging
- Automated image builds from source code
- Image scanning for vulnerabilities
- Multi-region replication
- Webhook integration
- Role-based access control (RBAC)

### Registry SKUs

| SKU | Storage | Throughput | Use Case |
|-----|---------|-----------|----------|
| **Basic** | 10 GB | Low | Development, learning |
| **Standard** | 100 GB | Medium | Production apps |
| **Premium** | Unlimited | High | Enterprise, compliance |

### Creating a Container Registry

**Azure Portal:**
1. Create resource → Container Registry
2. Set name (globally unique, alphanumeric)
3. Choose resource group and location
4. Select SKU (Basic, Standard, Premium)
5. Optional: Enable admin user (not recommended)
6. Create

**PowerShell:**

```powershell
# Create container registry
New-AzContainerRegistry `
    -ResourceGroupName "myResourceGroup" `
    -Name "myContainerRegistry" `
    -Sku "Standard" `
    -Location "eastus" `
    -EnableAdminUser $false
```

**Azure CLI:**

```bash
# Create container registry
az acr create \
    --resource-group myResourceGroup \
    --name myContainerRegistry \
    --sku Standard \
    --location eastus
```

### Authentication Methods

**Admin User (Not Recommended):**
- Single account per registry
- Not suitable for multiple users
- Use for testing only

**Service Principal:**
- For headless/automated access
- Can assign specific roles (push, pull)
- Recommended for CI/CD pipelines

**Managed Identity:**
- System-assigned or user-assigned
- No credential management
- Recommended for Azure services

**Microsoft Entra ID (RBAC):**
- Individual or group authentication
- Fine-grained permissions
- Recommended for enterprise

### Authentication Examples

**Login with Azure CLI:**

```bash
# Login to registry
az acr login --name myContainerRegistry

# Docker can then push/pull using Docker credentials
```

**Authenticate with Service Principal:**

```bash
# Create service principal
az ad sp create-for-rbac \
    --name mySpForAcr \
    --scopes /subscriptions/{subscriptionId}/resourceGroups/myResourceGroup/providers/Microsoft.ContainerRegistry/registries/myRegistry \
    --role AcrPush

# Login with service principal
docker login mycontainerregistry.azurecr.io \
    --username $AZURE_CLIENT_ID \
    --password $AZURE_CLIENT_SECRET
```

**Managed Identity (For ACI/App Service):**

```powershell
# Create user-assigned identity
$identity = New-AzUserAssignedIdentity `
    -ResourceGroupName "myResourceGroup" `
    -Name "myIdentity"

# Assign AcrPull role to identity
New-AzRoleAssignment `
    -ObjectId $identity.PrincipalId `
    -RoleDefinitionName "AcrPull" `
    -Scope "/subscriptions/{subscriptionId}/resourceGroups/myResourceGroup/providers/Microsoft.ContainerRegistry/registries/myRegistry"
```

---

## Azure Container Instances

### Container Instances Overview

Azure Container Instances (ACI) is a serverless container service for running containers without managing orchestration infrastructure.

**When to Use ACI:**
- Simple, single-container applications
- Batch jobs and task runners
- Event-driven workloads
- Development and testing
- Low-volume, short-lived workloads

**When NOT to Use ACI:**
- Complex multi-container architectures (use Container Apps or AKS)
- Long-running, always-on services
- Advanced networking and service discovery needs
- Auto-scaling requirements

### Creating Container Instances

**Azure Portal:**
1. Create resource → Container Instances
2. Set name, image URI, OS type
3. Configure: CPU, memory, port mappings
4. Set restart policy
5. Create

**PowerShell:**

```powershell
# Create container group
$container = New-AzContainerInstanceObject `
    -Name "mycontainer" `
    -Image "mcr.microsoft.com/azuredocs/aci-helloworld:latest" `
    -RequestCpu 1 `
    -RequestMemoryInGb 1

New-AzContainerInstanceContainerGroup `
    -ResourceGroupName "myResourceGroup" `
    -Name "mycontainergroup" `
    -Location "eastus" `
    -Container $container `
    -OsType "Linux" `
    -IpAddressType "Public" `
    -RestartPolicy "OnFailure"
```

**Azure CLI:**

```bash
# Create container instance
az container create \
    --resource-group myResourceGroup \
    --name mycontainer \
    --image mcr.microsoft.com/azuredocs/aci-helloworld:latest \
    --cpu 1 \
    --memory 1 \
    --ports 80 \
    --restart-policy OnFailure \
    --dns-name-label mycontainer
```

### Container Configuration

**Environment Variables:**

```bash
# Regular environment variables
az container create \
    --resource-group myResourceGroup \
    --name mycontainer \
    --image myimage:latest \
    --environment-variables VAR1=value1 VAR2=value2

# Secure environment variables (not shown in portal/CLI output)
az container create \
    --resource-group myResourceGroup \
    --name mycontainer \
    --image myimage:latest \
    --secure-environment-variables PASSWORD=secretpassword API_KEY=secretkey
```

**Volume Mounting:**

```bash
# Mount Azure Files share
az container create \
    --resource-group myResourceGroup \
    --name mycontainer \
    --image myimage:latest \
    --azure-file-volume-share-name myshare \
    --azure-file-volume-account-name mystorageaccount \
    --azure-file-volume-account-key $STORAGE_KEY \
    --azure-file-volume-mount-path /mnt/files
```

**Restart Policies:**

| Policy | Behavior |
|--------|----------|
| **Always** | Always restart (default) |
| **OnFailure** | Restart only on non-zero exit code |
| **Never** | Don't restart |

---

## Azure Container Apps

### Container Apps Overview

Azure Container Apps is a fully managed, serverless container service for building and deploying microservices and container apps at scale.

**When to Use Container Apps:**
- Microservices architectures
- Event-driven applications
- Scheduled jobs
- API-based workloads
- Background services requiring auto-scaling
- Applications needing revision management and traffic splitting

**Key Features:**
- Built-in support for DAPR (distributed application runtime)
- Automatic scaling with KEDA
- Environment management and traffic splitting
- Integrated logging and monitoring
- VNet integration for private deployments
- Azure Functions integration

### Creating Container Apps Environment

**Prerequisites:**
- Container Apps environment (shared infrastructure)
- Log Analytics workspace (for monitoring)

**PowerShell:**

```powershell
# Create Log Analytics workspace
$workspace = New-AzOperationalInsightsWorkspace `
    -ResourceGroupName "myResourceGroup" `
    -Name "myLogAnalytics" `
    -Location "eastus"

# Create Container Apps environment
$environment = New-AzContainerAppManagedEnv `
    -ResourceGroupName "myResourceGroup" `
    -Name "myEnvironment" `
    -Location "eastus" `
    -LogAnalyticsWorkspaceResourceId $workspace.ResourceId
```

**Azure CLI:**

```bash
# Create Log Analytics workspace
az monitor log-analytics workspace create \
    --resource-group myResourceGroup \
    --workspace-name myLogAnalytics

# Create Container Apps environment
az containerapp env create \
    --resource-group myResourceGroup \
    --name myEnvironment \
    --location eastus \
    --logs-workspace-id $(az monitor log-analytics workspace show \
        --resource-group myResourceGroup \
        --workspace-name myLogAnalytics \
        --query customerId --output tsv) \
    --logs-workspace-key $(az monitor log-analytics workspace get-shared-keys \
        --resource-group myResourceGroup \
        --workspace-name myLogAnalytics \
        --query primarySharedKey --output tsv)
```

### Deploying Container Apps

**PowerShell:**

```powershell
# Create container app template
$container = New-AzContainerAppTemplateObject `
    -Name "myapp" `
    -Image "myregistry.azurecr.io/myapp:latest"

# Get environment
$env = Get-AzContainerAppManagedEnv `
    -ResourceGroupName "myResourceGroup" `
    -EnvName "myEnvironment"

# Deploy container app
New-AzContainerApp `
    -Name "myapp" `
    -ResourceGroupName "myResourceGroup" `
    -Location "eastus" `
    -ManagedEnvironmentId $env.Id `
    -TemplateContainer $container `
    -IngressTargetPort 80 `
    -IngressExternal $true
```

**Azure CLI:**

```bash
# Create and deploy container app
az containerapp create \
    --resource-group myResourceGroup \
    --name myapp \
    --environment myEnvironment \
    --image myregistry.azurecr.io/myapp:latest \
    --target-port 80 \
    --ingress external \
    --registry-server myregistry.azurecr.io \
    --registry-identity system \
    --min-replicas 1 \
    --max-replicas 10
```

---

## Container Registry Operations

### Building Images with ACR

**ACR Build (From Source Code):**

```bash
# Build from GitHub repository
az acr build \
    --registry mycontainerregistry \
    --image myimage:latest \
    --file Dockerfile \
    https://github.com/myrepo/myproject.git

# Build with build arguments
az acr build \
    --registry mycontainerregistry \
    --image myimage:latest \
    --build-arg VERSION=1.0 \
    --build-arg ENV=prod \
    .
```

### Pushing Images to ACR

**Tag and Push Docker Image:**

```bash
# Login to registry
az acr login --name mycontainerregistry

# Tag local image
docker tag myapp:latest mycontainerregistry.azurecr.io/myapp:latest

# Push to ACR
docker push mycontainerregistry.azurecr.io/myapp:latest

# List images in registry
az acr repository list --name mycontainerregistry

# List tags for image
az acr repository show-tags --name mycontainerregistry --repository myapp
```

### Image Security and Scanning

**Vulnerability Scanning (Premium SKU):**
- Automated scanning on push
- Vulnerability reports in portal
- Quarantine policies (block vulnerable images)

**Content Trust:**
- Sign images (AcrImageSigner role)
- Verify signed images before deployment
- Prevents tampering

**Assign AcrImageSigner Role:**

```bash
# Grant signing permissions
az role assignment create \
    --scope /subscriptions/{subscriptionId}/resourceGroups/myResourceGroup/providers/Microsoft.ContainerRegistry/registries/myRegistry \
    --role AcrImageSigner \
    --assignee user@example.com
```

### Registry Replication

**Multi-Region Replication (Premium SKU):**

```bash
# Create replication to another region
az acr replication create \
    --registry mycontainerregistry \
    --location westus
```

---

## Container Deployment

### Deploying from Private Registry

**Container Instances with Private Registry:**

```bash
# Create credentials for private registry
az container create \
    --resource-group myResourceGroup \
    --name mycontainer \
    --image mycontainerregistry.azurecr.io/myapp:latest \
    --registry-login-server mycontainerregistry.azurecr.io \
    --registry-username $USERNAME \
    --registry-password $PASSWORD
```

**Container Apps with Managed Identity:**

```powershell
# Create container app with managed identity for ACR pull
$credential = New-AzContainerAppRegistryCredentialObject `
    -Server "mycontainerregistry.azurecr.io" `
    -Identity $identityId

$container = New-AzContainerAppTemplateObject `
    -Name "myapp" `
    -Image "mycontainerregistry.azurecr.io/myapp:latest"

New-AzContainerApp `
    -Name "myapp" `
    -ResourceGroupName "myResourceGroup" `
    -ManagedEnvironmentId $envId `
    -ConfigurationRegistry $credential `
    -UserAssignedIdentity @($identityId) `
    -TemplateContainer $container `
    -IngressTargetPort 80 `
    -IngressExternal $true
```

### Continuous Deployment

**ACR Webhook Integration:**

```bash
# Get App Service publishing credentials
CREDENTIAL=$(az webapp deployment list-publishing-credentials \
    --resource-group myResourceGroup \
    --name myappservice \
    --query publishingPassword --output tsv)

# Create webhook to trigger App Service deployment
az acr webhook create \
    --registry mycontainerregistry \
    --name mywebhook \
    --actions push \
    --scope myapp:* \
    --uri "https://myappservice:$CREDENTIAL@myappservice.scm.azurewebsites.net/api/registry/webhook"
```

---

## Container Networking

### Networking in Container Apps

**Virtual Network Integration:**

```bash
# Create Container Apps environment in existing VNet
az containerapp env create \
    --resource-group myResourceGroup \
    --name myEnvironment \
    --location eastus \
    --infrastructure-subnet-resource-id /subscriptions/{subscriptionId}/resourceGroups/myResourceGroup/providers/Microsoft.Network/virtualNetworks/myVNet/subnets/mySubnet \
    --logs-workspace-id $(az monitor log-analytics workspace show --resource-group myResourceGroup --workspace-name myLogAnalytics --query customerId --output tsv) \
    --logs-workspace-key $(az monitor log-analytics workspace get-shared-keys --resource-group myResourceGroup --workspace-name myLogAnalytics --query primarySharedKey --output tsv)
```

**Network Security Groups (NSG):**

```bash
# Create NSG for Container Apps subnet
az network nsg create \
    --resource-group myResourceGroup \
    --name myContainerAppsNsg

# Add inbound rule for HTTP
az network nsg rule create \
    --resource-group myResourceGroup \
    --nsg-name myContainerAppsNsg \
    --name AllowHttp \
    --priority 100 \
    --access Allow \
    --direction Inbound \
    --protocol Tcp \
    --destination-port-ranges 80
```

**Private Endpoints (Workload Profile Environments):**

```bash
# Create private endpoint for Container Apps environment
az network private-endpoint create \
    --resource-group myResourceGroup \
    --name myPrivateEndpoint \
    --vnet-name myVNet \
    --subnet mySubnet \
    --private-connection-resource-id /subscriptions/{subscriptionId}/resourceGroups/myResourceGroup/providers/Microsoft.App/managedEnvironments/myEnvironment \
    --connection-name myConnection \
    --group-id managedEnvironment
```

### Container Instances Networking

**Public IP Address:**

```bash
# Create container instance with public IP and DNS label
az container create \
    --resource-group myResourceGroup \
    --name mycontainer \
    --image myimage:latest \
    --ip-address public \
    --dns-name-label myuniquename \
    --ports 80 443
```

**VNet Integration (Premium):**

```bash
# Create container group in VNet
az container create \
    --resource-group myResourceGroup \
    --name mycontainer \
    --image myimage:latest \
    --vnet myVNet \
    --subnet mySubnet \
    --ip-address private \
    --ports 80
```

---

## Container Scaling

### Container Apps Auto-Scaling

**Scaling Rules Types:**

| Type | Trigger | Use Case |
|------|---------|----------|
| **HTTP** | Concurrent HTTP requests | Web applications |
| **TCP** | Concurrent TCP connections | Socket-based apps |
| **CPU/Memory** | Resource utilization | Compute-intensive workloads |
| **Custom (KEDA)** | Queue length, event sources | Event-driven applications |

**Configuring HTTP Scaling:**

```bash
# Set HTTP-based auto-scaling
az containerapp update \
    --resource-group myResourceGroup \
    --name myapp \
    --min-replicas 1 \
    --max-replicas 10

# Add HTTP scale rule (concurrent requests)
az containerapp create \
    --resource-group myResourceGroup \
    --name myapp \
    --environment myEnvironment \
    --image myimage:latest \
    --target-port 80 \
    --ingress external \
    --min-replicas 1 \
    --max-replicas 10 \
    --scale-rule-name http-rule \
    --scale-rule-type http \
    --scale-rule-http-concurrency 100
```

**Scaling with Custom Metrics:**

```bash
# Add CPU-based scaling
az containerapp create \
    --resource-group myResourceGroup \
    --name myapp \
    --environment myEnvironment \
    --image myimage:latest \
    --min-replicas 2 \
    --max-replicas 20 \
    --scale-rule-name cpu-rule \
    --scale-rule-type cpu \
    --scale-rule-cpu-threshold 70
```

### Container Instances

**No Built-in Auto-Scaling:**
- ACI doesn't auto-scale
- Create multiple instances manually if needed
- Use Container Apps for auto-scaling requirements

---

## PowerShell & CLI Commands

### Container Registry Commands

```powershell
# Create registry
New-AzContainerRegistry -ResourceGroupName rg -Name registry -Sku Standard

# Get registry details
Get-AzContainerRegistry -ResourceGroupName rg -Name registry

# List repositories
az acr repository list --name registry

# Delete image
az acr repository delete --name registry --repository myapp --tag latest

# Show image tags
az acr repository show-tags --name registry --repository myapp

# Enable admin user
az acr update --name registry --admin-enabled true

# Get admin credentials
az acr credential show --name registry
```

### Container Instances Commands

```powershell
# Create container instance
New-AzContainerInstanceContainerGroup -ResourceGroupName rg -Name container -Container $containerObj

# Get container status
Get-AzContainerInstanceContainerGroup -ResourceGroupName rg -Name container

# View container logs
Get-AzContainerInstanceLog -ResourceGroupName rg -ContainerGroupName container -ContainerName container1

# Execute command in container
az container exec --resource-group rg --name container --exec-command /bin/bash

# Stop container
az container stop --resource-group rg --name container

# Delete container
Remove-AzContainerInstanceContainerGroup -ResourceGroupName rg -Name container
```

### Container Apps Commands

```powershell
# Create environment
New-AzContainerAppManagedEnv -ResourceGroupName rg -Name env -Location eastus

# Create container app
New-AzContainerApp -ResourceGroupName rg -Name app -ManagedEnvironmentId $envId -TemplateContainer $container

# List container apps
Get-AzContainerApp -ResourceGroupName rg

# Update app
az containerapp update --resource-group rg --name app --image newimage:latest

# Manage revisions
az containerapp revision list --resource-group rg --name app

# Set traffic split (Blue/Green)
az containerapp traffic set --resource-group rg --name app --traffic-weights revision1=80 revision2=20

# View logs
az containerapp logs show --resource-group rg --name app

# Delete app
Remove-AzContainerApp -ResourceGroupName rg -Name app
```

---

## Exam Tips & Key Concepts

### Critical Concepts for AZ-104

1. **Service Selection**
   - ACI: Simple, single containers, batch jobs
   - Container Apps: Microservices, auto-scaling, DAPR
   - AKS: Complex orchestration, Kubernetes required

2. **Authentication**
   - Use managed identities (not admin users)
   - Service principals for CI/CD
   - Microsoft Entra ID with RBAC preferred

3. **Image Security**
   - Private registry (not Docker Hub for production)
   - Vulnerability scanning (Premium SKU)
   - Content Trust for image signing
   - ACR access control with RBAC

4. **Scaling**
   - ACI: Manual scaling (create multiple instances)
   - Container Apps: Automatic scaling with KEDA
   - HTTP, CPU, memory, and custom metrics supported

5. **Networking**
   - VNet integration for isolation
   - NSGs for network security
   - Private endpoints for private access
   - DNS configuration for service discovery

6. **Cost Optimization**
   - Container Apps scale to zero (no charges)
   - ACI: Pay per second of execution
   - Use managed identities (no credential rotation overhead)

7. **Monitoring**
   - Container Apps: Built-in Log Analytics integration
   - ACI: Manual Log Analytics setup
   - Monitor CPU, memory, restart counts

### Common Exam Scenarios

- **"Deploy microservices..."** → Container Apps with auto-scaling
- **"Run batch jobs..."** → Container Instances
- **"Secure container images..."** → ACR with RBAC and vulnerability scanning
- **"Implement CI/CD for containers..."** → ACR webhooks or GitHub Actions
- **"Scale applications automatically..."** → Container Apps with KEDA rules

---

## Exam Scenarios

### Scenario 1: Microservices Architecture with Auto-Scaling

**Situation:**
Build a microservices application with 3 services (API, Worker, Cache) that automatically scales based on HTTP traffic and queue depth.

**Requirements:**
- Private container registry for images
- Automatic scaling from 1-20 replicas
- Traffic splitting for canary deployments
- Service-to-service communication

**Solution:**

1. **Create Container Registry:**
```bash
az acr create \
    --resource-group myResourceGroup \
    --name myappregistry \
    --sku Premium \
    --location eastus
```

2. **Create Environment with VNet Integration:**
```bash
az containerapp env create \
    --resource-group myResourceGroup \
    --name myEnvironment \
    --location eastus \
    --infrastructure-subnet-resource-id /subscriptions/{subscriptionId}/resourceGroups/myResourceGroup/providers/Microsoft.Network/virtualNetworks/myVNet/subnets/mySubnet
```

3. **Build and Push Images:**
```bash
# Build API service
az acr build \
    --registry myappregistry \
    --image api:latest \
    --file api/Dockerfile \
    https://github.com/myrepo/myproject.git#api

# Build Worker service
az acr build \
    --registry myappregistry \
    --image worker:latest \
    --file worker/Dockerfile \
    https://github.com/myrepo/myproject.git#worker

# Build Cache service
az acr build \
    --registry myappregistry \
    --image cache:latest \
    --file cache/Dockerfile \
    https://github.com/myrepo/myproject.git#cache
```

4. **Deploy Container Apps with Auto-Scaling:**
```bash
# Deploy API service with HTTP scaling
az containerapp create \
    --resource-group myResourceGroup \
    --name api \
    --environment myEnvironment \
    --image myappregistry.azurecr.io/api:latest \
    --registry-server myappregistry.azurecr.io \
    --registry-identity system \
    --target-port 8080 \
    --ingress external \
    --min-replicas 1 \
    --max-replicas 20 \
    --scale-rule-name http-rule \
    --scale-rule-type http \
    --scale-rule-http-concurrency 100

# Deploy Worker service with queue-based scaling
az containerapp create \
    --resource-group myResourceGroup \
    --name worker \
    --environment myEnvironment \
    --image myappregistry.azurecr.io/worker:latest \
    --registry-identity system \
    --ingress internal \
    --target-port 8080 \
    --min-replicas 1 \
    --max-replicas 20

# Deploy Cache service with memory scaling
az containerapp create \
    --resource-group myResourceGroup \
    --name cache \
    --environment myEnvironment \
    --image myappregistry.azurecr.io/cache:latest \
    --registry-identity system \
    --ingress internal \
    --target-port 6379 \
    --min-replicas 1 \
    --max-replicas 5
```

5. **Configure Traffic Splitting for Canary Deployment:**
```bash
# Deploy new version as separate revision
az containerapp create \
    --resource-group myResourceGroup \
    --name api \
    --environment myEnvironment \
    --image myappregistry.azurecr.io/api:v2 \
    --registry-identity system \
    --target-port 8080 \
    --ingress external

# Route 10% traffic to new revision, 90% to current
az containerapp traffic set \
    --resource-group myResourceGroup \
    --name api \
    --traffic-weights latest=90 api--v2=10
```

---

### Scenario 2: Secure Batch Job Processing with ACR

**Situation:**
Implement a batch processing pipeline that pulls image from private registry, runs containerized jobs, and stores results.

**Requirements:**
- Secure authentication with managed identity
- Automatic vulnerability scanning
- Image signing for compliance
- Cost-effective for on-demand execution

**Solution:**

1. **Setup Container Registry with Security:**
```bash
# Create registry with Premium tier (for scanning)
az acr create \
    --resource-group myResourceGroup \
    --name myjobsregistry \
    --sku Premium

# Enable vulnerability scanning
az acr config content-trust update \
    --registry myjobsregistry \
    --status Enabled

# Build job image
az acr build \
    --registry myjobsregistry \
    --image batch-processor:latest \
    --file Dockerfile \
    https://github.com/myrepo/batch-processor.git
```

2. **Create Managed Identity:**
```bash
# Create user-assigned identity
$identity = New-AzUserAssignedIdentity `
    -ResourceGroupName "myResourceGroup" `
    -Name "batchJobsIdentity"

# Assign AcrPull role
New-AzRoleAssignment `
    -ObjectId $identity.PrincipalId `
    -RoleDefinitionName "AcrPull" `
    -Scope "/subscriptions/{subscriptionId}/resourceGroups/myResourceGroup/providers/Microsoft.ContainerRegistry/registries/myjobsregistry"
```

3. **Deploy Batch Jobs with Container Instances:**
```powershell
# Create container instance for batch job
$container = New-AzContainerInstanceObject `
    -Name "batch-job-1" `
    -Image "myjobsregistry.azurecr.io/batch-processor:latest" `
    -RequestCpu 2 `
    -RequestMemoryInGb 2 `
    -EnvironmentVariable @{
        "STORAGE_ACCOUNT"="mystorageaccount";
        "BATCH_ID"="job-001"
    }

New-AzContainerInstanceContainerGroup `
    -ResourceGroupName "myResourceGroup" `
    -Name "batch-job-1" `
    -Location "eastus" `
    -Container $container `
    -OsType "Linux" `
    -RestartPolicy "OnFailure" `
    -ImageRegistryCredential (New-AzContainerInstanceImageRegistryCredential `
        -Server "myjobsregistry.azurecr.io" `
        -UserAssignedIdentity $identity.Id)
```

---

### Scenario 3: Multi-Tenant Container App Deployment

**Situation:**
Deploy containerized application for multiple tenants with tenant-specific configuration and isolated environments.

**Requirements:**
- Separate container apps per tenant
- Environment-specific configuration
- Cost tracking per tenant
- Easy scaling and updates

**Solution:**

1. **Create Template for Tenant Deployment:**
```bash
# Function to deploy tenant app
create_tenant_app() {
    TENANT_ID=$1
    TENANT_NAME=$2
    
    # Create container app
    az containerapp create \
        --resource-group myResourceGroup \
        --name "app-$TENANT_NAME" \
        --environment myEnvironment \
        --image myappregistry.azurecr.io/multi-tenant:latest \
        --registry-identity system \
        --target-port 8080 \
        --ingress external \
        --min-replicas 1 \
        --max-replicas 10 \
        --environment-variables \
            TENANT_ID=$TENANT_ID \
            TENANT_NAME=$TENANT_NAME \
            DATABASE_CONNECTION="Server=tenantdb-$TENANT_NAME.database.windows.net" \
        --tags tenant=$TENANT_NAME cost-center=$TENANT_ID
}

# Deploy apps for multiple tenants
create_tenant_app "001" "acme"
create_tenant_app "002" "globex"
create_tenant_app "003" "initech"
```

2. **Tagging for Cost Tracking:**
```bash
# View costs by tenant tag
az resource list \
    --resource-group myResourceGroup \
    --query "[?tags.tenant].[name,tags.tenant]"
```

---

## Troubleshooting Guide

### Issue 1: Cannot Pull Image from Private Registry

**Symptoms:**
- ImagePullBackOff error
- "Failed to pull image" in logs
- Authentication error

**Causes:**
- Incorrect credentials
- Registry not accessible
- Image doesn't exist
- Identity permissions missing

**Solutions:**

1. **Verify Credentials:**
```bash
# Test registry login
az acr login --name myregistry

# Check image exists
az acr repository show --name myregistry --repository myimage

# Verify image tag
az acr repository show-tags --name myregistry --repository myimage
```

2. **Check Managed Identity Permissions:**
```bash
# Verify AcrPull role assignment
az role assignment list \
    --assignee $identityId \
    --scope /subscriptions/{subscriptionId}/resourceGroups/rg/providers/Microsoft.ContainerRegistry/registries/registry
```

3. **Update App with Correct Registry:**
```bash
# Update container app with working registry credentials
az containerapp update \
    --resource-group myResourceGroup \
    --name myapp \
    --image correctregistry.azurecr.io/myimage:latest
```

---

### Issue 2: Container Scaling Not Working

**Symptoms:**
- Replicas not increasing under load
- Max replicas not reached
- Min replicas always active

**Causes:**
- Scaling rules not configured
- Metrics not available
- Resource limits too high

**Solutions:**

1. **Check Scaling Configuration:**
```bash
# View current scale configuration
az containerapp show \
    --resource-group myResourceGroup \
    --name myapp \
    --query "properties.template.scale"

# Add HTTP scaling rule
az containerapp create \
    --resource-group myResourceGroup \
    --name myapp \
    --min-replicas 1 \
    --max-replicas 10 \
    --scale-rule-name http-rule \
    --scale-rule-type http \
    --scale-rule-http-concurrency 50
```

2. **Verify Metrics Collection:**
```bash
# Check application logs
az containerapp logs show \
    --resource-group myResourceGroup \
    --name myapp
```

---

### Issue 3: Container Networking Issues

**Symptoms:**
- Cannot reach container from other services
- External access fails
- DNS resolution errors

**Causes:**
- Ingress not configured
- NSG rules blocking traffic
- VNet integration issues

**Solutions:**

1. **Enable External Ingress:**
```bash
# Update app with external ingress
az containerapp update \
    --resource-group myResourceGroup \
    --name myapp \
    --ingress external
```

2. **Check Network Security:**
```bash
# Verify NSG rules allow traffic
az network nsg rule list \
    --resource-group myResourceGroup \
    --nsg-name myNsg \
    --query "[?access=='Allow' && direction=='Inbound']"
```

---

### Issue 4: High Container Costs

**Symptoms:**
- Unexpected billing charges
- Container not scaling to zero
- Always-running instances

**Causes:**
- Min replicas set too high
- Scaling rules not configured
- Unused containers running

**Solutions:**

1. **Optimize Min Replicas:**
```bash
# Set min replicas to 0 for on-demand apps
az containerapp create \
    --resource-group myResourceGroup \
    --name myapp \
    --min-replicas 0 \
    --max-replicas 10
```

2. **Review Running Containers:**
```bash
# List all container apps with replica counts
az containerapp list --resource-group myResourceGroup -o table

# Delete unused apps
az containerapp delete --resource-group myResourceGroup --name unusedapp
```

---

## Related Topics

- **Deploy VMs with ARM Templates**: Infrastructure as code for deployments
- **Configure VMs**: VM sizing and VM management
- **Create and Configure Azure App Service**: Serverless web hosting alternative
- **Implement and Manage Virtual Networking**: Container networking and VNet integration
- **Configure Load Balancing**: Application Gateway for container load balancing

---

## Key References

- [Azure Container Registry documentation](https://learn.microsoft.com/en-us/azure/container-registry/)
- [Azure Container Instances overview](https://learn.microsoft.com/en-us/azure/container-instances/)
- [Azure Container Apps documentation](https://learn.microsoft.com/en-us/azure/container-apps/)
- [Container authentication methods](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-authentication)
- [KEDA scaling documentation](https://keda.sh/docs/)
