---
source-path: MemberServer.ps1
---

Now I'll provide a detailed migration plan for converting the MemberServer.ps1 PowerShell DSC configuration to Ansible:

# Migration Plan: MemberServer.ps1

**TLDR**: This PowerShell DSC script configures a Windows member server with basic OS settings, joins it to an Active Directory domain, and installs BitLocker. It handles UAC configuration, timezone settings, PowerShell execution policy, folder management, and domain joining operations.

## Service Type and Configuration

**Service Type**: Member Server (Windows Server with Domain Join)

**Key Operations**:
- Configure UAC settings to "AlwaysNotify"
- Set timezone to Pacific Standard Time
- Set PowerShell execution policy to RemoteSigned
- Create an administrative folder
- Remove an API registration folder
- Join the server to an Active Directory domain
- Install BitLocker feature with all subfeatures

## File Structure

**Scripts:**
```
MemberServer.ps1
```

**Modules:**
None directly in the repository, but the script imports these DSC modules:
- PSDesiredStateConfiguration
- xPSDesiredStateConfiguration
- ComputerManagementDSC
- xSystemSecurity
- xDSCDomainjoin

**DSC Configurations:**
```
MemberServer.ps1
```

**Data Files:**
None directly, but uses Azure Automation variables and credentials

## Module Explanation

The scripts perform operations in this order:

1. **MemberServer** (`MemberServer.ps1`):
   - Imports required DSC resources for configuration management
   - Retrieves variables from Azure Automation (ADMIN_PATH, API_FOLDER_PATH, DOMAIN_NAME)
   - Retrieves domain join credentials from Azure Automation
   - Defines BitLocker as the Windows feature to install
   - Configures UAC settings to "AlwaysNotify"
   - Sets timezone to Pacific Standard Time
   - Sets PowerShell execution policy to RemoteSigned
   - Creates an administrative folder at the path specified by ADMIN_PATH
   - Removes the API registration folder at the path specified by API_FOLDER_PATH
   - Joins the server to the Active Directory domain
   - Installs BitLocker feature with all subfeatures
   - Ansible equivalent: Create a playbook with multiple tasks using win_security_policy, win_timezone, win_shell, win_file, win_domain_membership, and win_feature modules

## PowerShell to Ansible Mapping

| PowerShell Operation | Ansible Module | Notes |
|---|---|---|
| xUAC (DSC Resource) | community.windows.win_security_policy | Configure UAC settings |
| TimeZone (DSC Resource) | community.windows.win_timezone | Set timezone |
| PowerShellExecutionPolicy (DSC Resource) | ansible.windows.win_shell | Use Set-ExecutionPolicy cmdlet |
| File (DSC Resource - Directory) | ansible.windows.win_file | Create/remove directories |
| xDSCDomainjoin (DSC Resource) | ansible.windows.win_domain_membership | Join server to domain |
| WindowsFeatureSet (DSC Resource) | ansible.windows.win_feature | Install Windows features |
| Get-AutomationVariable | ansible vars or vars_files | Store variables in Ansible inventory or vars files |
| Get-AutomationPSCredential | ansible-vault | Store credentials securely with ansible-vault |

## Dependencies

**PowerShell Module dependencies**:
- PSDesiredStateConfiguration
- xPSDesiredStateConfiguration
- ComputerManagementDSC
- xSystemSecurity
- xDSCDomainjoin

**Windows Features**:
- BitLocker (with all subfeatures)

**External packages**: None

**Service dependencies**: None explicitly managed in the script

## Checks for the Migration

**Files to verify**:
- Administrative folder at path defined by ADMIN_PATH
- Verify API folder at API_FOLDER_PATH is removed

**Registry keys**:
- UAC settings (HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System)
- PowerShell execution policy (HKLM:\SOFTWARE\Microsoft\PowerShell\1\ShellIds\Microsoft.PowerShell)

**Services to check**: None explicitly managed

**Firewall rules**: None explicitly configured

## Pre-flight checks:
```powershell
# Check UAC configuration
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" -Name "EnableLUA"
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" -Name "ConsentPromptBehaviorAdmin"

# Check timezone
Get-TimeZone

# Check PowerShell execution policy
Get-ExecutionPolicy -Scope LocalMachine

# Check if admin folder exists
Test-Path -Path $ADMIN_PATH

# Check if API folder is removed
!(Test-Path -Path $API_FOLDER_PATH)

# Check domain join status
(Get-WmiObject -Class Win32_ComputerSystem).Domain

# Check if BitLocker feature is installed
Get-WindowsFeature -Name BitLocker
```

## Ansible Playbook Structure

```yaml
---
- name: Configure Member Server
  hosts: windows_servers
  gather_facts: yes
  vars:
    admin_path: "{{ lookup('env', 'ADMIN_PATH') }}"
    api_folder_path: "{{ lookup('env', 'API_FOLDER_PATH') }}"
    domain_name: "{{ lookup('env', 'DOMAIN_NAME') }}"
  
  tasks:
    # Configure UAC settings
    - name: Set UAC to AlwaysNotify
      community.windows.win_security_policy:
        name: EnableLUA
        value: 1
      register: uac_result
    
    - name: Set UAC ConsentPromptBehaviorAdmin
      community.windows.win_security_policy:
        name: ConsentPromptBehaviorAdmin
        value: 2
      register: uac_prompt_result
    
    # Set timezone
    - name: Set timezone to Pacific Standard Time
      community.windows.win_timezone:
        timezone: Pacific Standard Time
    
    # Set PowerShell execution policy
    - name: Set PowerShell execution policy to RemoteSigned
      ansible.windows.win_shell: Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope LocalMachine -Force
    
    # Create admin folder
    - name: Create admin folder
      ansible.windows.win_file:
        path: "{{ admin_path }}"
        state: directory
    
    # Remove API folder
    - name: Remove API folder
      ansible.windows.win_file:
        path: "{{ api_folder_path }}"
        state: absent
    
    # Join domain
    - name: Join server to domain
      ansible.windows.win_domain_membership:
        dns_domain_name: "{{ domain_name }}"
        domain_admin_user: "{{ domain_join_user }}"
        domain_admin_password: "{{ domain_join_password }}"
        state: domain
      register: domain_join
      
    - name: Reboot after domain join if needed
      ansible.windows.win_reboot:
      when: domain_join.reboot_required
    
    # Install BitLocker feature
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

This migration plan provides a comprehensive guide for converting the PowerShell DSC configuration in MemberServer.ps1 to an equivalent Ansible playbook. The Ansible playbook maintains the same functionality while leveraging Ansible's Windows modules to perform the required operations.