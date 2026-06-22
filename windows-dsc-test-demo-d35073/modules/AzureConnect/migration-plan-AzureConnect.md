---
source-path: AzureConnect.ps1
---

Now I'll create a detailed migration plan for converting the AzureConnect.ps1 PowerShell DSC script to Ansible:

# Migration Plan: AzureConnect.ps1

**TLDR**: This script configures basic Azure VM settings using PowerShell DSC. It sets up UAC, timezone, PowerShell execution policy, creates/removes specific folders, joins a domain, and installs BitLocker. The script is designed to connect Azure VMs to an on-premises Active Directory domain with basic security configurations.

## Service Type and Configuration

**Service Type**: Azure VM Configuration / Domain Join

**Key Operations**:
- Configure User Account Control (UAC) settings
- Set timezone to Pacific Standard Time
- Configure PowerShell execution policy to RemoteSigned
- Create an administrative folder
- Remove an API registration folder
- Join the VM to an Active Directory domain
- Install BitLocker feature with all subfeatures

## File Structure

**Scripts:**
```
AzureConnect.ps1
```

**DSC Configurations:**
```
AzureConnect.ps1
```

## Module Explanation

The script performs operations in this order:

1. **AzureConnect** (`AzureConnect.ps1`):
   - Imports required DSC resources from multiple modules
   - Retrieves variables and credentials from Azure Automation
   - Configures UAC settings to "AlwaysNotify"
   - Sets timezone to Pacific Standard Time
   - Sets PowerShell execution policy to RemoteSigned
   - Creates an admin folder at path stored in $ADMIN_PATH
   - Removes API folder at path stored in $API_FOLDER_PATH
   - Joins the VM to Active Directory domain using stored credentials
   - Installs BitLocker feature with all subfeatures
   - Ansible equivalent: Multiple Ansible modules in a playbook

## PowerShell to Ansible Mapping

| PowerShell Operation | Ansible Module | Notes |
|---|---|---|
| xUAC (DSC Resource) | community.windows.win_security_policy | Configure UAC settings |
| TimeZone (DSC Resource) | community.windows.win_timezone | Set timezone |
| PowerShellExecutionPolicy (DSC Resource) | community.windows.win_powershell | Set execution policy |
| File (DSC Resource - Directory) | ansible.windows.win_file | Create/remove directories |
| xDSCDomainjoin (DSC Resource) | community.windows.win_domain_membership | Join domain |
| WindowsFeatureSet (DSC Resource) | ansible.windows.win_feature | Install Windows features |
| Get-AutomationVariable | ansible variables | Store in vars section or vars_files |
| Get-AutomationPSCredential | ansible-vault | Store credentials securely |

## Dependencies

**PowerShell Module dependencies**:
- PSDesiredStateConfiguration
- xPSDesiredStateConfiguration
- ComputerManagementDSC
- xSystemSecurity
- xDSCDomainjoin

**Windows Features**:
- BitLocker

**External packages**: None

**Service dependencies**: None

## Checks for the Migration

**Files to verify**:
- Admin folder at path stored in $ADMIN_PATH
- API folder at path stored in $API_FOLDER_PATH (should be removed)

**Registry keys**:
- UAC settings registry keys
- PowerShell execution policy registry keys

**Services to check**:
- BitLocker service

**Firewall rules**: None specified

## Pre-flight checks:
```powershell
# Check if BitLocker is installed
Get-WindowsFeature -Name BitLocker

# Check timezone
Get-TimeZone

# Check PowerShell execution policy
Get-ExecutionPolicy -Scope LocalMachine

# Check domain join status
systeminfo | findstr /B "Domain"

# Check UAC settings
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" -Name "EnableLUA"
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" -Name "ConsentPromptBehaviorAdmin"
```

## Ansible Playbook Structure

Here's how to structure your Ansible playbook to replace the PowerShell DSC configuration:

```yaml
---
- name: Configure Azure VM and Join Domain
  hosts: windows_servers
  gather_facts: yes
  vars:
    admin_path: "{{ lookup('env', 'ADMIN_PATH') }}"
    api_folder_path: "{{ lookup('env', 'API_FOLDER_PATH') }}"
    domain_name: "{{ lookup('env', 'DOMAIN_NAME') }}"
  vars_files:
    - vault.yml  # Contains encrypted credentials

  tasks:
    - name: Configure UAC settings
      community.windows.win_security_policy:
        name: EnableLUA
        value: 1
      register: uac_result

    - name: Configure UAC prompt behavior for admins
      community.windows.win_security_policy:
        name: ConsentPromptBehaviorAdmin
        value: 2  # AlwaysNotify

    - name: Set timezone to Pacific Standard Time
      community.windows.win_timezone:
        timezone: Pacific Standard Time

    - name: Set PowerShell execution policy to RemoteSigned
      community.windows.win_powershell:
        execution_policy: RemoteSigned
        scope: LocalMachine

    - name: Create Admin folder
      ansible.windows.win_file:
        path: "{{ admin_path }}"
        state: directory

    - name: Remove API Registration folder
      ansible.windows.win_file:
        path: "{{ api_folder_path }}"
        state: absent
        force: yes

    - name: Join domain
      community.windows.win_domain_membership:
        dns_domain_name: "{{ domain_name }}"
        hostname: "{{ ansible_hostname }}"
        domain_admin_user: "{{ domain_join_user }}"
        domain_admin_password: "{{ domain_join_password }}"
        state: domain
      register: domain_join
      
    - name: Reboot after domain join if needed
      ansible.windows.win_reboot:
      when: domain_join.reboot_required

    - name: Install BitLocker feature
      ansible.windows.win_feature:
        name: BitLocker
        state: present
        include_sub_features: yes
        include_management_tools: yes
      register: feature_install

    - name: Reboot after feature installation if needed
      ansible.windows.win_reboot:
      when: feature_install.reboot_required
```

The vault.yml file should contain:
```yaml
# Encrypted with ansible-vault
domain_join_user: username@domain.com
domain_join_password: your_secure_password
```

This playbook follows the same logical flow as the original PowerShell DSC script while using Ansible's idempotent approach to configuration management.