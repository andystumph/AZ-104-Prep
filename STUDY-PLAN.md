# AZ-104 10-Week Study Plan

> **Comprehensive week-by-week study schedule to prepare for AZ-104 exam**
> 
> **Total Study Time:** ~30-40 hours | **Practice Time:** ~10-15 hours | **Total Duration:** 10 weeks

---

## Overview

This study plan is designed to help you systematically prepare for the AZ-104 exam by following a structured schedule that aligns with exam weight distribution. Each week includes:
- **Reading assignments** (study material from this repository)
- **Hands-on practice** (Azure portal, PowerShell, CLI)
- **Knowledge check** (self-assessment)
- **Time estimate** (hours per week)

---

## Week 1-2: Identity & Governance Foundation (3-4 hours)

### Focus: Understanding Azure identity and RBAC fundamentals

**Week 1: Microsoft Entra (Azure AD) & RBAC Basics**

| Day | Task | Time | Resource |
|-----|------|------|----------|
| Mon | Read: Users, groups, and licenses | 1.5h | [Manage AAD Objects](Manage%20Azure%20Identities%20and%20Governance/Manage%20AAD%20Objects.md) |
| Tue | Hands-on: Create users, groups in Azure portal | 1h | Azure Portal + Quick-Reference-Guide |
| Wed | Read: RBAC fundamentals and scope hierarchy | 1.5h | [Manage RBAC](Manage%20Azure%20Identities%20and%20Governance/Manage%20RBAC.md) |
| Thu | Hands-on: Assign roles at different scopes | 1h | PowerShell: `New-AzRoleAssignment` |
| Fri | Practice: Role assignment scenarios | 1h | Review exam scenario 3+ from RBAC file |
| Sat | Knowledge check: RBAC quiz | 0.5h | [Quick-Reference-Guide](Quick-Reference-Guide.md) |
| Sun | Rest/Review | - | Review notes, prepare Week 2 |

**Week 2: Governance, Policy & Subscriptions**

| Day | Task | Time | Resource |
|-----|------|------|----------|
| Mon | Read: Azure Policy and resource locks | 1.5h | [Manage Subscriptions and Governance](Manage%20Azure%20Identities%20and%20Governance/Manage%20Subscriptions%20and%20Governance.md) |
| Tue | Hands-on: Create policy definition, apply lock | 1.5h | Azure Portal + PowerShell |
| Wed | Read: Tags, management groups, cost management | 1.5h | Same file, sections 3-5 |
| Thu | Hands-on: Tag resources, explore Azure Advisor | 1h | Azure Portal + CLI |
| Fri | Practice: Governance scenario walkthrough | 1h | Review scenario 5 from Subscriptions file |
| Sat | Mock quiz: Identity & Governance | 0.5h | [Quick-Reference-Guide](Quick-Reference-Guide.md) - Concepts section |
| Sun | Review weak areas + Buffer | 0.5h | Reread difficult topics |

**Knowledge Check (End of Week 2):**
- [ ] Understand user/group management and SSPR
- [ ] Can explain RBAC scope hierarchy (subscription, RG, resource)
- [ ] Know when to use Azure Policy vs locks
- [ ] Understand management group hierarchy and inheritance

**Hands-On Completed:**
- [ ] Created users, groups, and licenses in Entra
- [ ] Assigned roles at 3+ different scopes
- [ ] Applied resource locks (CanNotDelete, ReadOnly)
- [ ] Created and assigned Azure Policy
- [ ] Tagged resources and created management group

---

## Week 2-3: Storage (3-4 hours)

### Focus: Storage accounts, redundancy, security, files, and blobs

| Day | Task | Time | Resource |
|-----|------|------|----------|
| Mon | Read: Storage account creation, redundancy options | 1.5h | [Secure Storage](Implement%20and%20Manage%20Storage/Secure%20Storage.md) - Intro + Redundancy |
| Tue | Hands-on: Create storage account, test GRS failover | 1.5h | Azure Portal + CLI: `az storage account create` |
| Wed | Read: SAS tokens, access control, firewalls | 1.5h | Same file, sections 3-4 |
| Thu | Hands-on: Generate SAS token, configure firewall | 1h | PowerShell: `New-AzStorageAccountSASToken` |
| Fri | Read: Azure Files and Blob Storage configuration | 1.5h | [Configure Azure Files and Blob Storage](Implement%20and%20Manage%20Storage/Configure%20Azure%20Files%20and%20Blob%20Storage.md) |
| Sat | Hands-on: Create file share, blob container, lifecycle policy | 1.5h | Portal + CLI |
| Sun | Practice: Storage scenario + mock quiz | 1h | Review scenarios 3-4 from Storage files |

**Knowledge Check (End of Week 3):**
- [ ] Know all storage redundancy options (LRS, GRS, ZRS, GZRS)
- [ ] Can create and configure SAS tokens
- [ ] Understand difference between service and account-level SAS
- [ ] Know blob tiers (Hot, Cool, Archive) and when to use
- [ ] Can configure lifecycle management policies

**Hands-On Completed:**
- [ ] Created storage account with GRS redundancy
- [ ] Tested cross-region failover
- [ ] Generated and tested SAS tokens
- [ ] Created file share and blob container
- [ ] Configured soft delete, versioning, lifecycle policies
- [ ] Set up storage firewall and service endpoints

---

## Week 3-5: Compute Resources (5-6 hours)

### Focus: VMs, containers, App Service, and ARM templates (Highest exam weight!)

**Week 3: Virtual Machines (2 hours)**

| Day | Task | Time | Resource |
|-----|------|------|----------|
| Mon | Read: VM sizing, creation, configuration | 1.5h | [Configure VMs](Deploy%20and%20Manage%20Azure%20Compute%20Resources/Configure%20VMs.md) - Sections 1-3 |
| Tue | Hands-on: Create Windows and Linux VMs | 1.5h | Portal + PowerShell: `New-AzVM` |
| Wed | Read: Managed disks, encryption, availability | 1.5h | Same file, sections 4-6 |
| Thu | Hands-on: Add managed disk, enable encryption, resize VM | 1.5h | Portal + PowerShell |
| Fri | Read: Autoscaling and scale sets | 1.5h | Same file, sections 7-8 |
| Sat | Hands-on: Create VMSS, configure autoscale rules | 1.5h | Portal + PowerShell |
| Sun | Practice: VM scenarios + quiz | 1h | Review scenarios 3+ from VM file |

**Week 4: ARM Templates & Infrastructure as Code (2 hours)**

| Day | Task | Time | Resource |
|-----|------|------|----------|
| Mon | Read: ARM template structure, parameters, variables | 1.5h | [Deploy VMs with ARM Templates](Deploy%20and%20Manage%20Azure%20Compute%20Resources/Deploy%20VMs%20with%20ARM%20Templates.md) - Intro + Template Structure |
| Tue | Hands-on: Modify existing ARM template, deploy | 1.5h | Portal + CLI: `az deployment group create` |
| Wed | Read: Bicep syntax, linked templates | 1.5h | Same file, sections 3-4 |
| Thu | Hands-on: Create Bicep file, deploy to Azure | 1.5h | VS Code + PowerShell: `New-AzResourceGroupDeployment` |
| Fri | Read: Validation, what-if, template specs | 1.5h | Same file, sections 5-6 |
| Sat | Hands-on: Validate template, export from resource | 1h | Portal + CLI |
| Sun | Practice: ARM template scenarios | 0.5h | Review scenarios 3 from template file |

**Week 5: Containers & App Service (2 hours)**

| Day | Task | Time | Resource |
|-----|------|------|----------|
| Mon | Read: Container registries, ACI, ACA | 1.5h | [Create and Configure Containers](Deploy%20and%20Manage%20Azure%20Compute%20Resources/Create%20and%20Configure%20Containers.md) - Intro + ACR |
| Tue | Hands-on: Create container registry, push image | 1.5h | Portal + CLI: `az acr create`, `az acr build` |
| Wed | Read: Azure Container Instances | 1h | Same file, section 3 |
| Thu | Hands-on: Deploy container to ACI | 1h | Portal + CLI: `az container create` |
| Fri | Read: App Service plans, scaling, deployment slots | 1.5h | [Create and Configure Azure App Service](Deploy%20and%20Manage%20Azure%20Compute%20Resources/Create%20and%20Configure%20Azure%20App%20Service.md) - Sections 1-3 |
| Sat | Hands-on: Create App Service, configure custom domain, TLS | 1.5h | Portal + PowerShell |
| Sun | Practice: App Service scenarios + comprehensive quiz | 1h | Review scenarios from both files |

**Knowledge Check (End of Week 5):**
- [ ] Can explain VM sizing categories and when to use each
- [ ] Understand managed disks vs unmanaged disks
- [ ] Know disk encryption options (SSE, ADE, Encryption at Host)
- [ ] Can create and deploy ARM templates with parameters
- [ ] Understand Bicep syntax and advantages
- [ ] Can deploy containers to ACR, ACI, ACA
- [ ] Know App Service scaling options (vertical, horizontal, autoscale)

**Hands-On Completed:**
- [ ] Created VMs with custom sizes and configurations
- [ ] Enabled disk encryption and configured backup
- [ ] Created and deployed ARM templates
- [ ] Wrote Bicep files and validated syntax
- [ ] Created container registry and pushed images
- [ ] Deployed containers to ACI and ACA
- [ ] Created App Service with custom domain and TLS certificate
- [ ] Configured deployment slots and autoscaling

---

## Week 5-7: Virtual Networking (6-7 hours)

### Focus: VNets, peering, security, load balancing, hybrid connectivity (Second-highest weight!)

**Week 5-6: VNets, Subnets, Peering, Security (3-4 hours)**

| Day | Task | Time | Resource |
|-----|------|------|----------|
| Mon | Read: VNet/subnet creation, IP addressing, routing | 1.5h | [Implement and Manage Virtual Networking](Configure%20and%20Manage%20Virtual%20Networking/Implement%20and%20Manage%20Virtual%20Networking.md) |
| Tue | Hands-on: Create VNet, subnets, configure UDR | 1.5h | Portal + PowerShell: `New-AzVirtualNetwork` |
| Wed | Read: Network security groups, ASGs, Bastion | 1.5h | [Secure Access to Virtual Networks](Configure%20and%20Manage%20Virtual%20Networking/Secure%20Access%20to%20Virtual%20Networks.md) - Sections 1-3 |
| Thu | Hands-on: Create NSGs, define rules, test connectivity | 1.5h | Portal + CLI: `az network nsg create` |
| Fri | Read: Service endpoints, private endpoints | 1.5h | Same file, sections 4-5 |
| Sat | Hands-on: Configure service/private endpoints | 1.5h | Portal + PowerShell |
| Sun | Practice: Networking scenario 1-2 | 1h | Review scenarios from VNet file |

**Week 6-7: Load Balancing & Hybrid Connectivity (3-4 hours)**

| Day | Task | Time | Resource |
|-----|------|------|----------|
| Mon | Read: Azure Load Balancer, Application Gateway | 1.5h | [Configure Load Balancing](Configure%20and%20Manage%20Virtual%20Networking/Configure%20Load%20Balancing.md) - Sections 1-3 |
| Tue | Hands-on: Create load balancer, configure backend pools | 1.5h | Portal + PowerShell: `New-AzLoadBalancer` |
| Wed | Read: Traffic Manager, Front Door | 1.5h | Same file, sections 4-5 |
| Thu | Hands-on: Configure Traffic Manager, test failover | 1.5h | Portal + CLI |
| Fri | Read: VPN Gateway, ExpressRoute, Virtual WAN | 1.5h | [Integrate On-Prem Networks with Azure](Configure%20and%20Manage%20Virtual%20Networking/Integrate%20On-Prem%20Networks%20with%20Azure.md) - Intro + VPN Gateway |
| Sat | Hands-on: Create VPN gateway, configure S2S VPN | 1.5h | Portal + PowerShell |
| Sun | Practice: Hybrid connectivity scenarios + comprehensive quiz | 1h | Review all networking scenarios |

**Knowledge Check (End of Week 7):**
- [ ] Understand VNet address space and subnet planning
- [ ] Can create and manage NSGs with inbound/outbound rules
- [ ] Know difference between Azure Load Balancer (Layer 4) and Application Gateway (Layer 7)
- [ ] Understand Traffic Manager vs Front Door use cases
- [ ] Can configure VPN Gateway for S2S connectivity
- [ ] Know ExpressRoute peering types (private, Microsoft)
- [ ] Understand Virtual WAN hub-spoke topology

**Hands-On Completed:**
- [ ] Created multi-tier VNet with subnets and UDRs
- [ ] Configured NSGs with multiple security rules
- [ ] Implemented Azure Bastion for secure VM access
- [ ] Set up service and private endpoints
- [ ] Created load balancer with backend pools and health probes
- [ ] Configured Application Gateway with WAF
- [ ] Created and tested Traffic Manager failover
- [ ] Configured VPN Gateway with S2S connection
- [ ] (Optional) Set up ExpressRoute or Virtual WAN

---

## Week 7-8: Monitoring & Backup (3-4 hours)

### Focus: Azure Monitor, diagnostics, alerts, backup, disaster recovery

| Day | Task | Time | Resource |
|-----|------|------|----------|
| Mon | Read: Metrics, logs, diagnostic settings | 1.5h | [Monitor Resources with Azure Monitor](Monitor%20and%20Backup%20Azure%20Resources/Monitor%20Resources%20with%20Azure%20Monitor.md) - Sections 1-3 |
| Tue | Hands-on: Create metric alert, configure diagnostic settings | 1.5h | Portal + PowerShell: `Set-AzDiagnosticSetting` |
| Wed | Read: KQL, Log Analytics, alerts | 1.5h | Same file, sections 4-5 |
| Thu | Hands-on: Write KQL queries, create log alert | 1.5h | Log Analytics workspace + KQL queries |
| Fri | Read: Recovery Services Vault, backup policies | 1.5h | [Implement Backup and Recovery](Monitor%20and%20Backup%20Azure%20Resources/Implement%20Backup%20and%20Recovery.md) - Sections 1-3 |
| Sat | Hands-on: Create vault, enable VM backup, configure policies | 1.5h | Portal + PowerShell: `New-AzRecoveryServicesVault` |
| Sun | Read: Site Recovery, failover, soft delete | 1h | Same file, sections 7-10 |

**Week 8: Backup & Disaster Recovery Practice**

| Day | Task | Time | Resource |
|-----|------|------|----------|
| Mon | Hands-on: Configure SQL backup, test restore | 1.5h | PowerShell: Backup/Restore cmdlets |
| Tue | Read: Azure Site Recovery, RPO/RTO, test failover | 1h | Backup file, Site Recovery section |
| Wed | Hands-on: Enable Site Recovery replication, test failover | 1.5h | Portal + PowerShell |
| Fri | Practice: Backup/recovery scenarios | 1h | Review scenarios from Backup file |
| Sat | Comprehensive quiz: Monitoring & Backup | 0.5h | [Quick-Reference-Guide](Quick-Reference-Guide.md) |
| Sun | Rest/Review | - | Review all monitoring and backup topics |

**Knowledge Check (End of Week 8):**
- [ ] Can create metric and log alerts in Azure Monitor
- [ ] Know how to write KQL queries for log analysis
- [ ] Understand diagnostic settings and where data goes
- [ ] Can create Recovery Services vault and backup policies
- [ ] Know soft delete and immutable vault features
- [ ] Understand Site Recovery RPO/RTO concepts
- [ ] Can configure and test failover scenarios

**Hands-On Completed:**
- [ ] Created Azure Monitor metric and log alerts
- [ ] Configured diagnostic settings for VMs and storage
- [ ] Wrote KQL queries (CPU usage, failed logins, etc.)
- [ ] Created and tested VM backup, restore to alternate VM
- [ ] Configured SQL Server backup with retention policies
- [ ] Enabled Site Recovery replication
- [ ] Performed test failover and verified VM creation
- [ ] Configured soft delete and immutability for backup vault

---

## Week 8-10: Practice, Review & Exam Prep (10-15 hours)

### Focus: Reinforcement, weakness identification, exam simulation

**Week 8-9: Practice Tests & Targeted Review (8-10 hours)**

| Task | Time | Details |
|------|------|---------|
| **Official Microsoft Practice Test 1** | 2h | Take full exam simulation (40 questions, 100 min) |
| **Review wrong answers** | 1.5h | Study materials related to missed questions |
| **Domain-specific practice** | 2h | Focus on weakest domain (usually Compute or Networking) |
| **Official Microsoft Practice Test 2** | 2h | Second full exam simulation |
| **Final review of weak areas** | 1.5h | Concentrated study on specific topics |
| **Command practice drills** | 2h | Run all PowerShell/CLI commands from Quick-Reference-Guide |

**Target Scores:**
- [ ] Week 8: Practice Test 1 - Aim for 65-70%
- [ ] Week 9: Practice Test 2 - Aim for 75-80%
- [ ] Week 10: Practice Test 3 - Aim for 80%+

**Week 9-10: Final Review & Exam Day Prep (5-7 hours)**

| Task | Time | Details |
|------|------|---------|
| **Read Quick-Reference-Guide daily** | 30 min x 5 = 2.5h | 5-10 min review of different sections each day |
| **Review exam tips sections** | 1h | Go through all "Exam Tips" from each study file |
| **Final scenario practice** | 1.5h | Complete 5+ complex scenarios from study files |
| **Mock exam under test conditions** | 2h | Last practice test (simulate exam environment) |
| **Rest & mental prep** | Flexible | Get good sleep, avoid cramming last 2 days |

**Exam Day Checklist:**
- [ ] Schedule exam on day you feel most confident
- [ ] Arrive 15 minutes early
- [ ] Review NDA and test rules before starting
- [ ] Read each question carefully (watch for "NOT" keywords)
- [ ] Flag difficult questions and return later
- [ ] Leave 10-15 minutes for review at end
- [ ] Stay calm and trust your preparation

---

## Success Metrics

**Track Progress:**
- [ ] Complete all reading assignments
- [ ] Hands-on practice for each domain
- [ ] Pass all domain-specific quizzes (70%+)
- [ ] Score 75%+ on official practice tests
- [ ] Complete all major exam scenarios
- [ ] Comfortable explaining 5-10 complex topics

**Red Flags (Adjust Study Plan if Needed):**
- Scoring below 65% on practice tests after Week 8
- Unfamiliar with command syntax after Week 7
- Difficulty understanding hybrid networking concepts
- Consistently weak on disaster recovery/backup topics

---

## Exam Day Tips

1. **Time Management:** ~90 seconds per question; don't get stuck
2. **Read Carefully:** Multiple-choice has subtle differences
3. **Case Studies:** Read the scenario completely before answering related questions
4. **Performance-Based:** Lab questions usually require step-by-step configuration
5. **Review:** Use remaining time to review flagged questions
6. **Trust Preparation:** You've studied all domains thoroughly

---

## Post-Exam

**If You Pass 🎉**
- Congratulations! You're now an Azure Administrator!
- Share your experience and help others prepare
- Consider higher-level certifications (AZ-305 Architect Expert, etc.)

**If You Don't Pass**
- Review your exam feedback report
- Identify weak domains and re-study using this plan
- Focus on performance-based lab questions
- Reschedule within 24 hours while concepts are fresh
- Many people pass on second attempt with focused review

---

**Remember:** Consistency beats intensity. Studying 3-4 hours per week over 10 weeks is more effective than cramming 40 hours in the final week. You've got this! 🚀
