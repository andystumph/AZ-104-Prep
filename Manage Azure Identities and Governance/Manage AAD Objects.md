# Manage Microsoft Entra ID (Azure AD) Objects

> **Exam Weight:** Part of Manage Azure identities and governance (20-25%)

## Table of Contents
- [Overview](#overview)
- [Manage Microsoft Entra Users and Groups](#manage-microsoft-entra-users-and-groups)
- [Administrative Units](#administrative-units)
- [Licenses in Microsoft Entra ID](#licenses-in-microsoft-entra-id)
- [External Users (B2B)](#external-users-b2b)
- [Self-Service Password Reset (SSPR)](#self-service-password-reset-sspr)
- [PowerShell and CLI Commands](#powershell-and-cli-commands)
- [Best Practices](#best-practices)
- [Exam Tips](#exam-tips)

---

## Overview

**Microsoft Entra ID** (formerly Azure Active Directory) is Microsoft's cloud-based identity and access management service. It helps employees sign in and access resources.

### Key Concepts

- **Tenant**: An instance of Microsoft Entra ID representing an organization
- **Directory**: The collection of objects (users, groups, applications) in a tenant
- **Subscription**: A billing unit that can be associated with a tenant
- **Users**: Identities that can be authenticated
- **Groups**: Collections of users for easier permission management

### Microsoft Entra ID vs Active Directory DS

| Feature | Microsoft Entra ID | Active Directory DS |
|---------|-------------------|---------------------|
| Structure | Flat (no OUs) | Hierarchical (OUs, GPOs) |
| Protocol | REST API (HTTP/HTTPS) | LDAP, Kerberos |
| Authentication | SAML, OAuth, OpenID | Kerberos, NTLM |
| Devices | Azure AD Join, Hybrid Join | Domain Join |
| Management | Azure Portal, PowerShell, CLI | ADUC, PowerShell |

---

## Manage Microsoft Entra Users and Groups

### User Types

1. **Cloud-Only Users**: Created directly in Microsoft Entra ID
2. **Synced Users**: Synchronized from on-premises AD using Azure AD Connect
3. **Guest Users**: External users (B2B)

### Creating Users

#### Azure Portal
1. Navigate to **Microsoft Entra ID**
2. Select **Users** > **New user**
3. Choose **Create user** or **Invite external user**
4. Fill in required information:
   - User principal name (UPN)
   - Display name
   - Password settings
   - Job info (optional)
   - Groups and roles

#### PowerShell
```powershell
# Install Microsoft.Graph module
Install-Module Microsoft.Graph -Scope CurrentUser

# Connect to Microsoft Graph
Connect-MgGraph -Scopes "User.ReadWrite.All"

# Create a new user
$PasswordProfile = @{
    Password = "SecureP@ssw0rd123!"
    ForceChangePasswordNextSignIn = $true
}

New-MgUser `
    -DisplayName "Jane Doe" `
    -UserPrincipalName "jane.doe@contoso.com" `
    -MailNickname "janedoe" `
    -PasswordProfile $PasswordProfile `
    -AccountEnabled $true
```

#### Azure CLI
```bash
# Create a new user
az ad user create \
    --display-name "John Smith" \
    --password "SecureP@ssw0rd123!" \
    --user-principal-name "john.smith@contoso.com" \
    --force-change-password-next-sign-in true
```

### Managing User Properties

**Common properties to manage:**
- **Account status**: Enable/disable accounts
- **Job information**: Department, job title, manager
- **Contact information**: Phone numbers, addresses
- **Licenses**: Assign/remove licenses
- **Authentication methods**: MFA, password reset phone/email

#### Update User Properties (PowerShell)
```powershell
# Update user properties
Update-MgUser -UserId "jane.doe@contoso.com" `
    -Department "IT" `
    -JobTitle "Azure Administrator" `
    -MobilePhone "+1234567890"
```

### Groups in Microsoft Entra ID

#### Group Types

1. **Security Groups**: Used for access control
   - Can contain users, devices, groups, and service principals
   - Used for RBAC assignments

2. **Microsoft 365 Groups**: Used for collaboration
   - Includes shared mailbox, calendar, files, SharePoint site
   - Can only contain users

#### Group Assignment Types

1. **Assigned**: Manual membership management
2. **Dynamic User**: Auto-membership based on user properties
3. **Dynamic Device**: Auto-membership based on device properties

#### Creating Security Groups (PowerShell)
```powershell
# Create an assigned security group
New-MgGroup `
    -DisplayName "Azure Administrators" `
    -MailEnabled $false `
    -SecurityEnabled $true `
    -MailNickname "azureadmins" `
    -GroupTypes @()

# Create a dynamic user group
$groupParams = @{
    DisplayName = "Sales Department"
    MailEnabled = $false
    SecurityEnabled = $true
    MailNickname = "salesdept"
    GroupTypes = @("DynamicMembership")
    MembershipRule = "(user.department -eq ""Sales"")"
    MembershipRuleProcessingState = "On"
}
New-MgGroup @groupParams
```

#### Dynamic Group Rules Examples

```
# Users in the Sales department
(user.department -eq "Sales")

# Users in London or Paris
(user.city -eq "London") -or (user.city -eq "Paris")

# Users whose job title contains "Manager"
(user.jobTitle -contains "Manager")

# Devices running Windows
(device.deviceOSType -eq "Windows")

# Users with specific extension attribute
(user.extensionAttribute1 -eq "VIP")
```

### Bulk Operations

#### Bulk User Creation
1. Download the CSV template from Azure Portal
2. Fill in user information
3. Upload the CSV file
4. Monitor the bulk operation status

**CSV Example:**
```csv
version:v1.0
Name,User name,Initial password,Block sign in (Yes/No)
John Smith,john.smith@contoso.com,Pass@word123,No
Jane Doe,jane.doe@contoso.com,Pass@word123,No
```

#### Bulk Operations with PowerShell
```powershell
# Import users from CSV
$users = Import-Csv -Path "users.csv"

foreach ($user in $users) {
    $PasswordProfile = @{
        Password = $user.Password
        ForceChangePasswordNextSignIn = $true
    }
    
    New-MgUser `
        -DisplayName $user.DisplayName `
        -UserPrincipalName $user.UserPrincipalName `
        -MailNickname $user.MailNickname `
        -PasswordProfile $PasswordProfile `
        -AccountEnabled $true
}
```

---

## Administrative Units

**Administrative Units** enable you to delegate administrative permissions to specific subsets of users, groups, or devices within your organization.

### Key Features
- Restrict administrative scope to a portion of your organization
- Useful for large, decentralized organizations
- Requires **Microsoft Entra ID P1 or P2** license

### Use Cases
- Geographic delegation (e.g., US Admin, EU Admin)
- Department delegation (e.g., Sales Admin, IT Admin)
- Business unit delegation

### Creating Administrative Units

#### Azure Portal
1. Navigate to **Microsoft Entra ID** > **Administrative units**
2. Select **New administrative unit**
3. Provide name and description
4. Add members (users, groups, devices)
5. Assign roles scoped to the AU

#### PowerShell
```powershell
# Create an administrative unit
New-MgDirectoryAdministrativeUnit `
    -DisplayName "North America" `
    -Description "Administrative unit for North American users"

# Add users to administrative unit
$auId = (Get-MgDirectoryAdministrativeUnit -Filter "displayName eq 'North America'").Id
$userId = (Get-MgUser -Filter "userPrincipalName eq 'john@contoso.com'").Id

New-MgDirectoryAdministrativeUnitMemberByRef `
    -AdministrativeUnitId $auId `
    -BodyParameter @{"@odata.id" = "https://graph.microsoft.com/v1.0/users/$userId"}
```

### Dynamic Administrative Units
Like dynamic groups, you can create administrative units with automatic membership based on user/device properties.

```powershell
$auParams = @{
    DisplayName = "US Users"
    MembershipType = "Dynamic"
    MembershipRule = "(user.country -eq ""United States"")"
    MembershipRuleProcessingState = "On"
}
New-MgDirectoryAdministrativeUnit @auParams
```

---

## Licenses in Microsoft Entra ID

### License Types
- **Microsoft 365** (E3, E5, Business Premium)
- **Microsoft Entra ID** (P1, P2)
- **Azure AD Free** (included with Azure subscription)
- **Individual service licenses** (Exchange, SharePoint, Teams)

### License Assignment Methods

1. **Direct Assignment**: Assign to individual users
2. **Group-Based Licensing**: Assign to groups (recommended)
   - Requires Microsoft Entra ID P1 or P2
   - Automatically assigns/removes licenses based on group membership

### Managing Licenses

#### Azure Portal
1. Navigate to **Microsoft Entra ID** > **Licenses**
2. Select **All products**
3. Choose a product
4. Select **Assign**
5. Choose users or groups

#### PowerShell
```powershell
# Assign license to user
Set-MgUserLicense `
    -UserId "john@contoso.com" `
    -AddLicenses @{SkuId = "SkuId-GUID"} `
    -RemoveLicenses @()

# Assign license via group-based licensing
$groupId = (Get-MgGroup -Filter "displayName eq 'Azure Admins'").Id

Set-MgGroupLicense `
    -GroupId $groupId `
    -AddLicenses @{SkuId = "SkuId-GUID"} `
    -RemoveLicenses @()
```

### License Service Plans
Each license includes multiple service plans. You can disable specific services:

```powershell
# Get available SKUs
Get-MgSubscribedSku | Select-Object SkuPartNumber, SkuId, ServicePlans

# Assign license with disabled service plans
$disabledPlans = @("ServicePlanId1", "ServicePlanId2")
Set-MgUserLicense `
    -UserId "john@contoso.com" `
    -AddLicenses @{
        SkuId = "SkuId-GUID"
        DisabledPlans = $disabledPlans
    } `
    -RemoveLicenses @()
```

---

## External Users (B2B)

**Azure AD B2B** (Business-to-Business) allows you to invite external users to access your resources.

### Guest User Characteristics
- External email address as UPN
- UserType = "Guest"
- Limited permissions by default
- Can be added to groups and assigned roles

### Inviting Guest Users

#### Azure Portal
1. Navigate to **Microsoft Entra ID** > **Users**
2. Select **New user** > **Invite external user**
3. Enter email address
4. Add personal message (optional)
5. Assign groups/roles
6. Send invitation

#### PowerShell
```powershell
# Invite guest user
New-MgInvitation `
    -InvitedUserEmailAddress "external@partner.com" `
    -InviteRedirectUrl "https://portal.azure.com" `
    -SendInvitationMessage $true `
    -InvitedUserDisplayName "External Partner"
```

### Managing External Collaboration Settings

Configure external collaboration settings in **External Identities** > **External collaboration settings**:

- Who can invite guests (admins only, users, etc.)
- Collaboration restrictions (allow/block domains)
- Guest user permissions
- Guest access restrictions

### Guest User Permissions

**Default restrictions:**
- Limited directory access
- Cannot enumerate users
- Cannot see groups they're not members of
- Cannot read most directory information

**Can be configured in:**
Microsoft Entra ID > Users > User settings > External users

---

## Self-Service Password Reset (SSPR)

**SSPR** allows users to reset their own passwords without contacting IT support.

### Requirements
- Microsoft Entra ID P1 or P2 (or Microsoft 365 Business Premium)
- Users must be registered for SSPR
- Azure AD Connect password writeback (for hybrid scenarios)

### Enabling SSPR

#### Azure Portal
1. Navigate to **Microsoft Entra ID** > **Password reset**
2. Select **Properties**
3. Choose target scope:
   - **None**: SSPR disabled
   - **Selected**: Enabled for specific group
   - **All**: Enabled for all users
4. Configure authentication methods
5. Configure registration requirements

### Authentication Methods

**Available methods:**
- Mobile app notification
- Mobile app code
- Email
- Mobile phone (SMS/Call)
- Office phone
- Security questions

**Best practice**: Require at least 2 methods for reset

### Registration Requirements

Configure in **Password reset** > **Registration**:
- Require users to register when signing in
- Number of days before users must reconfirm
- Show "Contact your administrator" link

### Password Writeback

For hybrid environments, enable password writeback to sync password changes back to on-premises AD.

**Requirements:**
- Azure AD Connect
- Appropriate on-premises AD permissions
- Microsoft Entra ID P1 or P2

**Configure in Azure AD Connect:**
1. Run Azure AD Connect wizard
2. Select **Customize synchronization options**
3. Enable **Password writeback**
4. Complete wizard

### SSPR Policies

Configure additional policies:
- **On-premises integration**: Password writeback, account unlock
- **Notifications**: Notify users and admins on password resets
- **Customization**: Custom helpdesk link

---

## PowerShell and CLI Commands

### Essential PowerShell Commands (Microsoft.Graph)

```powershell
# Connect to Microsoft Graph
Connect-MgGraph -Scopes "User.ReadWrite.All", "Group.ReadWrite.All"

# User Management
Get-MgUser -UserId "john@contoso.com"
Get-MgUser -Filter "startsWith(displayName, 'John')"
New-MgUser -DisplayName "..." -UserPrincipalName "..." -PasswordProfile $profile
Update-MgUser -UserId "..." -Department "IT"
Remove-MgUser -UserId "..."

# Group Management
Get-MgGroup -Filter "displayName eq 'Azure Admins'"
New-MgGroup -DisplayName "..." -MailEnabled $false -SecurityEnabled $true
Add-MgGroupMember -GroupId "..." -DirectoryObjectId "..."
Get-MgGroupMember -GroupId "..."

# License Management
Get-MgUserLicenseDetail -UserId "..."
Set-MgUserLicense -UserId "..." -AddLicenses @{SkuId = "..."} -RemoveLicenses @()

# Guest Users
New-MgInvitation -InvitedUserEmailAddress "..." -InviteRedirectUrl "..."
Get-MgUser -Filter "userType eq 'Guest'"
```

### Essential Azure CLI Commands

```bash
# User Management
az ad user list
az ad user show --id john@contoso.com
az ad user create --display-name "..." --password "..." --user-principal-name "..."
az ad user update --id "..." --department "IT"
az ad user delete --id "..."

# Group Management
az ad group list
az ad group show --group "Azure Admins"
az ad group create --display-name "..." --mail-nickname "..."
az ad group member add --group "..." --member-id "..."
az ad group member list --group "..."

# Guest Users
az ad user get-member-groups --id "..."
```

---

## Best Practices

### User Management
1. **Use Naming Conventions**: Consistent UPN format (firstname.lastname@domain.com)
2. **Enable MFA**: Require multi-factor authentication for all users
3. **Regular Audits**: Review inactive accounts and remove them
4. **Least Privilege**: Grant minimum permissions necessary
5. **Separate Admin Accounts**: Use dedicated accounts for administrative tasks

### Group Management
1. **Use Dynamic Groups**: Automate membership management where possible
2. **Descriptive Names**: Use clear, consistent naming conventions
3. **Document Purpose**: Add descriptions to groups
4. **Group-Based Licensing**: Prefer group-based over direct assignment
5. **Regular Reviews**: Conduct access reviews for sensitive groups

### External Collaboration
1. **Restrict Invitations**: Limit who can invite guests
2. **Domain Whitelisting**: Allow only trusted partner domains
3. **Access Reviews**: Regular review guest user access
4. **Time-Limited Access**: Set expiration on guest accounts when possible
5. **MFA for Guests**: Require MFA for external users

### SSPR
1. **Require 2+ Methods**: Don't rely on single authentication method
2. **Force Registration**: Require registration at next sign-in
3. **Enable Writeback**: For hybrid environments
4. **Monitor Usage**: Review SSPR reports regularly
5. **User Education**: Train users on SSPR process

---

## Exam Tips

### Key Concepts to Remember
1. **User Types**: Cloud-only, synced, guest
2. **Group Types**: Security vs Microsoft 365
3. **Group Assignment**: Assigned vs Dynamic
4. **License Assignment**: Direct vs Group-based
5. **SSPR Requirements**: P1/P2 license, authentication methods
6. **Administrative Units**: Delegation at scale (requires P1/P2)

### Common Exam Scenarios
1. **Bulk user creation**: Know the CSV format and bulk operations
2. **Dynamic group rules**: Understand syntax and use cases
3. **Guest user permissions**: Know default restrictions and how to modify
4. **SSPR configuration**: Authentication methods, writeback, registration
5. **License conflicts**: Understand how to troubleshoot licensing issues
6. **Administrative delegation**: When to use Administrative Units vs RBAC

### Commands to Memorize
- `New-MgUser` / `az ad user create`
- `New-MgGroup` / `az ad group create`
- `New-MgInvitation` for B2B
- `Set-MgUserLicense` for licensing
- `New-MgDirectoryAdministrativeUnit` for AUs

### Watch Out For
- **Licensing requirements** for features (Free vs P1 vs P2)
- **Difference between Azure AD roles and Azure RBAC roles**
- **Dynamic group rules syntax** (operators, properties)
- **SSPR requires P1/P2** but not for all features
- **Guest users are external users** (UserType = Guest)
- **Administrative Units require P1/P2 licenses**

### Troubleshooting Scenarios
1. **User can't reset password**: Check SSPR enabled, authentication methods registered
2. **Group-based licensing not applying**: Check for license conflicts, enough licenses available
3. **Guest can't access resource**: Check guest user permissions, conditional access policies
4. **Dynamic group not populating**: Verify membership rule syntax, processing state
5. **Can't create administrative unit**: Verify P1/P2 licensing

---

## Related Topics
- [Manage RBAC](./Manage%20RBAC.md) - Role-Based Access Control
- [Manage Subscriptions and Governance](./Manage%20Subscriptions%20and%20Governance.md)
- [Official Microsoft Documentation](https://learn.microsoft.com/en-us/entra/identity/)

---

**Next Steps:**
1. Practice creating users and groups in Azure Portal
2. Set up a test tenant to practice bulk operations
3. Configure SSPR and test password reset flow
4. Create dynamic groups with different membership rules
5. Practice inviting and managing guest users
