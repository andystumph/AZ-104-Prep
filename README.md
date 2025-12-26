# AZ-104 Microsoft Azure Administrator - Exam Prep Study Guide

> **Comprehensive study materials for the AZ-104: Microsoft Azure Administrator certification exam**
> 
> Last Updated: December 2025 | Exam Version: April 18, 2025

## 📋 Exam Overview

The AZ-104 exam measures your ability to accomplish the following technical tasks:
- Manage Azure identities and governance (20-25%)
- Implement and manage storage (15-20%)
- Deploy and manage Azure compute resources (20-25%)
- Implement and manage virtual networking (15-20%)
- Monitor and maintain Azure resources (10-15%)

**Passing Score:** 700 out of 1000  
**Duration:** Approximately 120 minutes  
**Question Types:** Multiple choice, case studies, drag-and-drop, and hands-on labs

## 🎯 Prerequisites

Before taking this exam, you should have:
- Experience with operating systems, networking, servers, and virtualization
- Hands-on experience with PowerShell, Azure CLI, Azure Portal, and ARM templates
- Understanding of Microsoft Entra ID (formerly Azure AD)
- Basic understanding of cloud computing concepts

## 📚 Study Materials Organization

This repository organizes study materials by the five main exam domains. Each section contains detailed notes, commands, best practices, and exam tips.

### Domain Breakdown

#### 1. Manage Azure Identities and Governance (20-25%)
- [Manage Microsoft Entra (Azure AD) Objects](Manage%20Azure%20Identities%20and%20Governance/Manage%20AAD%20Objects.md)
- [Manage Role-Based Access Control (RBAC)](Manage%20Azure%20Identities%20and%20Governance/Manage%20RBAC.md)
- [Manage Subscriptions and Governance](Manage%20Azure%20Identities%20and%20Governance/Manage%20Subscriptions%20and%20Governance.md)

#### 2. Implement and Manage Storage (15-20%)
- [Secure Storage](Implement%20and%20Manage%20Storage/Secure%20Storage.md)
- [Manage Storage](Implement%20and%20Manage%20Storage/Manage%20Storage.md)
- [Configure Azure Files and Blob Storage](Implement%20and%20Manage%20Storage/Configure%20Azure%20Files%20and%20Blob%20Storage.md)

#### 3. Deploy and Manage Azure Compute Resources (20-25%)
- [Deploy VMs with ARM Templates and Bicep](Deploy%20and%20Manage%20Azure%20Compute%20Resources/Deploy%20VMs%20with%20ARM%20Templates.md)
- [Configure Virtual Machines](Deploy%20and%20Manage%20Azure%20Compute%20Resources/Configure%20VMs.md)
- [Create and Configure Containers](Deploy%20and%20Manage%20Azure%20Compute%20Resources/Create%20and%20Configure%20Containers.md)
- [Create and Configure Azure App Service](Deploy%20and%20Manage%20Azure%20Compute%20Resources/Create%20and%20Configure%20Azure%20App%20Service.md)

#### 4. Configure and Manage Virtual Networking (15-20%)
- [Implement and Manage Virtual Networks](Configure%20and%20Manage%20Virtual%20Networking/Implement%20and%20Manage%20Virtual%20Networking.md)
- [Secure Access to Virtual Networks](Configure%20and%20Manage%20Virtual%20Networking/Secure%20Access%20to%20Virtual%20Networks.md)
- [Configure Load Balancing](Configure%20and%20Manage%20Virtual%20Networking/Configure%20Load%20Balancing.md)
- [Monitor and Troubleshoot Virtual Networking](Configure%20and%20Manage%20Virtual%20Networking/Monitor%20and%20Troubleshoot%20Virtual%20Networking.md)
- [Integrate On-Premises Networks with Azure](Configure%20and%20Manage%20Virtual%20Networking/Integrate%20On-Prem%20Networks%20with%20Azure.md)

#### 5. Monitor and Maintain Azure Resources (10-15%)
- [Monitor Resources with Azure Monitor](Monitor%20and%20Backup%20Azure%20Resources/Monitor%20Resources%20with%20Azure%20Monitor.md) - Metrics, logs, alerts, monitoring insights
- [Implement Backup and Recovery](Monitor%20and%20Backup%20Azure%20Resources/Implement%20Backup%20and%20Recovery.md) - Backup policies, disaster recovery, Site Recovery

## 🚀 Quick Start Guide

**Recommended Study Path (10 weeks total):**

1. **Week 1-2: Identity & Governance** (3 files, ~3-4 hours)
   - Start with [Manage AAD Objects](Manage%20Azure%20Identities%20and%20Governance/Manage%20AAD%20Objects.md) to understand identity management
   - Foundation for all other exam domains

2. **Week 2-3: Storage** (3 files, ~3-4 hours)
   - Critical for data protection and compliance
   - Hands-on: Practice with storage accounts and containers

3. **Week 3-5: Compute Resources** (4 files, ~5-6 hours)
   - Highest exam weight (20-25%); allocate most study time
   - Hands-on: Create VMs, deploy containers, configure App Service

4. **Week 5-7: Virtual Networking** (5 files, ~6-7 hours)
   - Second-highest exam weight (15-20%)
   - Hands-on: Build VNets, configure NSGs, set up hybrid connectivity

5. **Week 7-8: Monitoring & Backup** (2 files, ~3-4 hours)
   - Hands-on: Create backups, configure alerts, test failover scenarios

6. **Week 8-10: Practice & Review**
   - Take practice tests (aim for 75%+ scores)
   - Review weak areas using [Quick-Reference-Guide](Quick-Reference-Guide.md)
   - Complete all hands-on scenarios from study files

## 💡 Study Tips

- **Hands-on experience is crucial** - The exam includes performance-based questions
- **Understand WHY, not just HOW** - Know when to use each service/feature
- **Master PowerShell and Azure CLI** - Many questions test command-line skills; use [Quick-Reference-Guide](Quick-Reference-Guide.md) for command syntax
- **Review case studies carefully** - Take time to understand requirements before answering
- **Use the exam sandbox** - Familiarize yourself with the exam interface beforehand
- **Practice commands daily** - Spend 30 minutes each day running PowerShell/CLI examples from study files
- **Take practice tests weekly** - Identify weak areas early and focus revision accordingly

## 🔗 Detailed Exam Objectives

## Manage Azure identities and governance (20–25%)

### Manage Microsoft Entra users and groups
- create users and groups
- manage user and group properties
- manage licenses in Microsoft Entra ID
- manage external users
- configure self-service password reset (SSPR)

### Manage access to Azure resources
- manage built-in Azure roles
- assign roles at different scopes
- interpret access assignments

### Manage Azure subscriptions and governance
- implement and manage Azure Policy
- configure resource locks
- apply and manage tags on resources
- manage resource groups
- manage subscriptions
- manage costs by using alerts, budgets, and Azure Advisor recommendations
- configure management groups

## Implement and manage storage (15–20%)

### Configure access to storage
- configure Azure Storage firewalls and virtual networks
- create and use shared access signature (SAS) tokens
- configure stored access policies
- manage access keys
- configure identity-based access for Azure Files

### Configure and manage storage accounts
- create and configure storage accounts
- configure Azure Storage redundancy
- configure object replication
- configure storage account encryption
- manage data by using Azure Storage Explorer and AzCopy

### Configure Azure files and Azure Blob Storage
- create and configure a file share in Azure Storage
- create and configure a container in Blob Storage
- configure storage tiers
- configure soft delete for blobs and containers
- configure snapshots and soft delete for Azure Files
- configure blob lifecycle management
- configure blob versioning

## Deploy and manage Azure compute resources (20–25%)

### Automate deployment of resources by using Azure Resource Manager (ARM) templates or Bicep files
- interpret an Azure Resource Manager template or a Bicep file
- modify an existing Azure Resource Manager template
- modify an existing Bicep file
- deploy resources by using an Azure Resource Manager template or a Bicep file
- export a deployment as an Azure Resource Manager template or convert an Azure Resource Manager template to a Bicep file

### Create and configure virtual machines
- create a virtual machine
- configure Azure Disk Encryption
- move a virtual machine to another resource group, subscription, or region
- manage virtual machine sizes
- manage virtual machine disks
- deploy virtual machines to availability zones and availability sets
- deploy and configure an Azure Virtual Machine Scale Sets

### Provision and manage containers in the Azure portal
- create and manage an Azure container registry
- provision a container by using Azure Container Instances
- provision a container by using Azure Container Apps
- manage sizing and scaling for containers, including Azure Container Instances and Azure Container Apps

### Create and configure Azure App Service
- provision an App Service plan
- configure scaling for an App Service plan
- create an App Service
- configure certificates and Transport Layer Security (TLS) for an App Service
- map an existing custom DNS name to an App Service
- configure backup for an App Service
- configure networking settings for an App Service
- configure deployment slots for an App Service

## Implement and manage virtual networking (15–20%)

### Configure and manage virtual networks in Azure
- create and configure virtual networks and subnets
- create and configure virtual network peering
- configure public IP addresses
- configure user-defined network routes
- troubleshoot network connectivity

### Configure secure access to virtual networks
- create and configure network security groups (NSGs) and application security groups
- evaluate effective security rules in NSGs
- implement Azure Bastion
- configure service endpoints for Azure platform as a service (PaaS)
- configure private endpoints for Azure PaaS

### Configure name resolution and load balancing
- configure Azure DNS
- configure an internal or public load balancer
- troubleshoot load balancing

## Monitor and maintain Azure resources (10–15%)

### Monitor resources in Azure
- interpret metrics in Azure Monitor
- configure log settings in Azure Monitor
- query and analyze logs in Azure Monitor
- set up alert rules, action groups, and alert processing rules in Azure Monitor
- configure and interpret monitoring of virtual machines, storage accounts, and networks by using Azure Monitor Insights
- use Azure Network Watcher and Connection Monitor

### Implement backup and recovery
- create a Recovery Services vault
- create an Azure Backup vault
- create and configure a backup policy
- perform backup and restore operations by using Azure Backup
- configure Azure Site Recovery for Azure resources
- perform a failover to a secondary region by using Site Recovery
- configure and interpret reports and alerts for backups

## 📖 Quick Links

**This Repository:**
- [Quick-Reference-Guide.md](Quick-Reference-Guide.md) - Commands, service limits, key concepts
- [PROJECT-STATUS.md](PROJECT-STATUS.md) - Detailed progress tracking and file descriptions
- [STUDY-PLAN.md](STUDY-PLAN.md) - Week-by-week study schedule with milestones

**Official Microsoft Resources:**
- [Microsoft Learn Exam AZ-104: Microsoft Azure Administrator](https://learn.microsoft.com/en-us/certifications/exams/az-104)
- [AZ-104 Exam Skills Measured](https://learn.microsoft.com/en-us/certifications/resources/study-guides/AZ-104)
- [Microsoft Azure Free Account](https://azure.microsoft.com/en-us/free/)

**Community Resources:**
- [John Savil's AZ-104 Azure Administrator Study List](https://www.youtube.com/watch?v=VOod_VNgdJk&list=PLlVtbbG169nGlGPWs9xaLKT1KfwqREHbs)
- [Pluralsight Microsoft Azure Administrator Associate](https://app.pluralsight.com/explore/certifications/topics/azure)
- [Microsoft Azure Practice Tests](https://aka.ms/az104practicetest)