---
source-path: ActiveDirectoryBuild.ps1
---

Based on the analysis of the PowerShell code, I'll now create a detailed migration plan for converting this PowerShell DSC configuration to Ansible.

# Migration Plan: ActiveDirectoryBuild.ps1

**TLDR**: This script configures an Active Directory Domain Controller, establishing a new forest with specific organizational units, groups, users, and DNS settings. It implements a tiered administrative model with security groups, creates service accounts, and sets up DNS forwarding rules. The script is designed to be run as a DSC configuration for establishing a secure Active Directory environment with proper delegation and security controls.

## Service Type and Configuration

**Service Type**: Active Directory Domain Services

**Key Operations**:
- Installs and configures Active Directory Domain Services and DNS
- Creates a new forest and domain
- Sets domain password and lockout policies
- Creates AD replication sites
- Establishes a tiered administrative model with security groups
- Creates organizational units for resources and managed devices
- Pre-stages Cluster Name Objects for various services
- Configures DNS conditional forwarders and default forwarders
- Creates service accounts and administrative users
- Sets up Group Managed Service Accounts (gMSAs)

## File Structure

**Scripts:**
```
ActiveDirectoryBuild.ps1
```

## Module Explanation

The scripts perform operations in this order:

1. **ActiveDirectoryBuild.ps1**:
   - Imports required DSC resources for Active Directory, DNS, and system management
   - Retrieves variables and credentials from Azure Automation
   - Installs Windows features for AD DS, DNS, and management tools
   - Configures base OS settings (UAC, timezone, PowerShell execution policy)
   - Creates a new Active Directory forest and domain
   - Sets domain password and lockout policies
   - Creates AD replication sites
   - Creates KDS root key for Group Managed Service Accounts
   - Creates security groups for tiered administration model
   - Creates organizational units for resources and managed devices
   - Pre-stages Cluster Name Objects for various services
   - Configures DNS conditional forwarders and default forwarders
   - Creates service accounts and administrative users
   - Ansible equivalent: Multiple Ansible roles and playbooks for AD deployment and configuration

## PowerShell to Ansible Mapping

| PowerShell Operation | Ansible Module | Notes |
|---|---|---|
| Configuration Block | ansible.builtin.include_role | Use roles for organizing related tasks |
| Import-DscResource | N/A | Ansible doesn't require explicit imports |
| Get-AutomationVariable | ansible.builtin.set_fact or vars files | Store variables in inventory, group_vars, or host_vars |
| Get-AutomationPSCredential | ansible.builtin.set_fact with vault | Use Ansible Vault for credential storage |
| xUAC | community.windows.win_security_policy | Configure UAC settings |
| TimeZone | community.windows.win_timezone | Set timezone |
| PowerShellExecutionPolicy | community.windows.win_powershell | Set PowerShell execution policy |
| File (Directory) | ansible.windows.win_file | Create or remove directories |
| WindowsFeatureSet | ansible.windows.win_feature | Install Windows features |
| ADDomain | community.windows.win_domain | Create new AD domain/forest |
| Service | ansible.windows.win_service | Manage Windows services |
| ADDomainDefaultPasswordPolicy | community.windows.win_domain_password_policy | Configure domain password policy |
| ADReplicationSite | community.windows.win_domain_controller | Configure AD replication sites |
| ADKDSKey | ansible.windows.win_shell | Use PowerShell commands for KDS key |
| ADGroup | community.windows.win_domain_group | Create and manage AD groups |
| ADOrganizationalUnit | community.windows.win_domain_ou | Create and manage OUs |
| ADComputer | community.windows.win_domain_computer | Create and manage computer objects |
| xDnsServerConditionalForwarder | community.windows.win_dns_zone | Configure DNS zones |
| xDnsServerForwarder | community.windows.win_dns_server | Configure DNS server settings |
| xDnsServerADZone | community.windows.win_dns_zone | Configure AD-integrated DNS zones |
| ADUser | community.windows.win_domain_user | Create and manage AD users |
| ADManagedServiceAccount | ansible.windows.win_shell | Use PowerShell commands for gMSA management |

## Dependencies

**PowerShell Module dependencies**:
- PSDesiredStateConfiguration
- xPSDesiredStateConfiguration
- ComputerManagementDSC
- xSystemSecurity
- ActiveDirectoryDsc
- xDnsServer

**Windows Features**:
- AD-Domain-Services
- DNS
- RSAT-AD-PowerShell
- RSAT-ADDS
- RSAT-DNS-Server
- BitLocker
- RSAT-Feature-Tools-BitLocker-BdeAducExt

**External packages**: None explicitly installed

**Service dependencies**:
- NTDS (Active Directory Domain Services)

## Checks for the Migration

**Files to verify**:
- Admin folder path (defined in variable $ADMIN_PATH)
- API folder path (defined in variable $API_FOLDER_PATH)

**Registry keys**: None explicitly modified

**Services to check**:
- NTDS (Active Directory Domain Services)

**Firewall rules**: None explicitly created

## Pre-flight checks:

```powershell
# Check if AD DS is installed
Get-WindowsFeature AD-Domain-Services

# Check if DNS is installed
Get-WindowsFeature DNS

# Check if domain exists
Get-ADDomain -Identity $DOMAIN_NAME

# Check if OUs exist
Get-ADOrganizationalUnit -Filter * | Where-Object {$_.Name -like "*TIER*"}

# Check if groups exist
Get-ADGroup -Filter * | Where-Object {$_.Name -like "Admin-*"}

# Check if users exist
Get-ADUser -Filter * | Where-Object {$_.Name -like "Admin*"}

# Check DNS configuration
Get-DnsServerForwarder
Get-DnsServerZone
```

## Ansible Implementation Approach

For implementing this in Ansible, I recommend creating a structured playbook with multiple roles:

1. **base_os_config**: Configure UAC, timezone, execution policy, and directories
2. **ad_install**: Install AD DS and DNS features
3. **ad_forest**: Create the forest and domain
4. **ad_policies**: Configure password and lockout policies
5. **ad_structure**: Create OUs, groups, and users
6. **ad_dns**: Configure DNS settings and forwarders
7. **ad_service_accounts**: Create and configure service accounts and gMSAs

Each role should have its own tasks, handlers, and defaults to make the configuration modular and reusable. Variables should be stored in group_vars or host_vars, with sensitive information in Ansible Vault.

The main playbook would include these roles in the correct order to ensure dependencies are met. For example, the AD forest must be created before OUs and users can be added.