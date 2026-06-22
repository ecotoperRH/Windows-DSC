---
source-path: ActiveDirectoryHub.ps1
---

Now I'll create a detailed migration plan for converting the ActiveDirectoryHub.ps1 PowerShell DSC configuration to Ansible.

# Migration Plan: ActiveDirectoryHub.ps1

**TLDR**: This script configures an Active Directory Domain Controller (hub) using PowerShell DSC. It installs necessary AD services and DNS, joins an existing domain as a domain controller, configures system settings like timezone and UAC, and monitors the NTDS service. The script uses credentials from Azure Automation and sets up directory structures.

## Service Type and Configuration

**Service Type**: Active Directory Domain Controller

**Key Operations**:
- Installs Active Directory Domain Services and related management tools
- Joins an existing domain as a domain controller
- Configures system settings (UAC, timezone, PowerShell execution policy)
- Creates and removes specific directories
- Monitors the NTDS service
- Uses credentials from Azure Automation

## File Structure

**Scripts:**
```
ActiveDirectoryHub.ps1
```

**Modules:**
None directly included, but imports several DSC modules:
- PSDesiredStateConfiguration
- xPSDesiredStateConfiguration
- ComputerManagementDSC
- xSystemSecurity
- ActiveDirectoryDsc

**DSC Configurations:**
```
ActiveDirectoryHub.ps1 (contains Configuration ActiveDirectoryHub)
```

**Data Files:**
None directly included, but uses Azure Automation variables

## Module Explanation

The scripts perform operations in this order:

1. **ActiveDirectoryHub.ps1**:
   - Imports required DSC resources for Active Directory, system security, and computer management
   - Retrieves variables and credentials from Azure Automation
   - Configures base OS settings (UAC, timezone, PowerShell execution policy)
   - Creates and removes specific directories
   - Installs Windows features for Active Directory and DNS
   - Waits for domain availability
   - Joins an existing domain as a domain controller
   - Monitors the NTDS service
   - Ansible equivalent: Multiple Ansible roles and playbooks for AD domain controller configuration

## PowerShell to Ansible Mapping

| PowerShell Operation | Ansible Module | Notes |
|---|---|---|
| Import-DscResource | N/A | Ansible uses modules directly without import statements |
| Get-AutomationVariable | ansible.builtin.set_fact or variables | Store in vars or use lookup plugins |
| Get-AutomationPSCredential | ansible.builtin.set_fact with vault | Use Ansible Vault for credentials |
| xUAC | community.windows.win_security_policy | Configure UAC settings |
| TimeZone | community.windows.win_timezone | Set timezone |
| PowerShellExecutionPolicy | community.windows.win_powershell | Set execution policy |
| File (Directory) | ansible.windows.win_file | Create/remove directories |
| WindowsFeatureSet | ansible.windows.win_feature | Install Windows features |
| WaitForADDomain | community.windows.win_domain_membership | Check domain availability |
| ADDomainController | community.windows.win_domain_controller | Configure domain controller |
| Service | ansible.windows.win_service | Manage Windows services |

## Dependencies

**PowerShell Module dependencies**:
- PSDesiredStateConfiguration
- xPSDesiredStateConfiguration
- ComputerManagementDSC
- xSystemSecurity
- ActiveDirectoryDsc

**Windows Features**:
- AD-Domain-Services
- DNS
- RSAT-AD-PowerShell
- RSAT-ADDS
- RSAT-DNS-Server
- BitLocker
- RSAT-Feature-Tools-BitLocker-BdeAducExt

**External packages**: None specified

**Service dependencies**:
- NTDS (Active Directory Domain Services)

## Checks for the Migration

**Files to verify**:
- Admin folder path (defined by $ADMIN_PATH variable)
- API folder path (defined by $API_FOLDER_PATH variable)

**Registry keys**: None explicitly modified

**Services to check**:
- NTDS (should be running and set to automatic startup)

**Firewall rules**: None explicitly created

## Pre-flight checks:

```yaml
# Verify domain controller status
- name: Check if Active Directory Domain Services is installed
  ansible.windows.win_feature_info:
    name: AD-Domain-Services
  register: ad_feature

- name: Check if server is a domain controller
  ansible.windows.win_shell: |
    (Get-WmiObject -Class Win32_ComputerSystem).DomainRole -ge 4
  register: dc_check

# Verify NTDS service status
- name: Check NTDS service status
  ansible.windows.win_service_info:
    name: NTDS
  register: ntds_service

# Verify domain membership
- name: Check domain membership
  ansible.windows.win_shell: |
    (Get-WmiObject -Class Win32_ComputerSystem).Domain
  register: domain_check
```

## Ansible Implementation Approach

Here's a step-by-step guide for implementing this in Ansible:

1. **Create variable files**:
   - Create a vars file or use Ansible Vault for sensitive credentials
   - Define variables for domain name and paths

2. **Create a playbook structure**:
   ```
   roles/
     ad_domain_controller/
       tasks/
         main.yml
         install_features.yml
         configure_os.yml
         join_domain.yml
         monitor_services.yml
       vars/
         main.yml
       handlers/
         main.yml
   ```

3. **Implement base OS configuration**:
   - Configure UAC, timezone, and PowerShell execution policy
   - Create and remove directories

4. **Install required features**:
   - Use win_feature to install AD services and management tools

5. **Configure domain controller**:
   - Wait for domain availability
   - Join as a domain controller

6. **Configure service monitoring**:
   - Ensure NTDS service is running and set to automatic

7. **Add handlers for service restarts**:
   - Create handlers for any required service restarts

8. **Create a main playbook**:
   - Include all the roles and tasks in the correct order
   - Add pre-flight checks and post-configuration validation

This migration will require careful handling of credentials, as the original script uses Azure Automation credentials which will need to be replaced with Ansible Vault or another secure credential management solution.