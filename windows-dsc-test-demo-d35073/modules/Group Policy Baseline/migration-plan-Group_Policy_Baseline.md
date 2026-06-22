---
source-path: group-policy-baseline
---

# Migration Plan: Group Policy Baseline

**TLDR**: This PowerShell script imports Group Policy Objects (GPOs) from a selected folder into Active Directory. It prompts the user to select a folder containing GPO backups, then iterates through each backup folder to import the GPOs with their original names as defined in the XML files.

## Service Type and Configuration

**Service Type**: Active Directory Group Policy Management

**Key Operations**:
- Imports Active Directory and GroupPolicy PowerShell modules
- Prompts user to select a folder containing GPO backups
- Reads GPO backup folders and their XML configuration files
- Imports each GPO with its original name as defined in the XML
- Creates GPOs if they don't already exist

## File Structure

**Scripts:**
- group-policy-baseline/ImportGPOBulk.ps1

**Modules:**
None (uses built-in Windows modules)

**DSC Configurations:**
None

**Data Files:**
None (GPO backup files are referenced but not part of the repository)

## Module Explanation

The scripts perform operations in this order:

1. **ImportGPOBulk.ps1** (`group-policy-baseline/ImportGPOBulk.ps1`):
   - Imports required PowerShell modules (ActiveDirectory and GroupPolicy)
   - Creates a file browser dialog to select the folder containing GPO backups
   - Gets a list of all items in the selected folder
   - For each item (GPO backup folder) in the folder:
     - Constructs the path to the gpreport.xml file
     - Reads the XML file to extract the original GPO name
     - Imports the GPO using the backup ID (folder name), original GPO name, and source path
     - Creates the GPO if it doesn't exist

## PowerShell to Ansible Mapping

| PowerShell Operation | Ansible Module | Notes |
|---|---|---|
| Import-Module ActiveDirectory | ansible.windows.win_shell or community.windows.win_powershell | Ansible will need to run PowerShell commands to interact with AD |
| Import-Module GroupPolicy | ansible.windows.win_shell or community.windows.win_powershell | Ansible will need to run PowerShell commands to interact with Group Policy |
| new-object -com Shell.Application | Not needed in Ansible | Ansible doesn't need interactive file browser |
| get-childitem | ansible.builtin.find | To list directories containing GPO backups |
| [XML](get-content $XMLFile) | ansible.builtin.xml | For parsing XML files |
| import-gpo | ansible.windows.win_shell or community.windows.win_powershell | Run the import-gpo command via PowerShell |

## Dependencies

**PowerShell Module dependencies**: ActiveDirectory, GroupPolicy  
**Windows Features**: RSAT-AD-PowerShell, GPMC (Group Policy Management Console)  
**External packages**: None  
**Service dependencies**: Active Directory Domain Services  

## Checks for the Migration

**Files to verify**: None (script doesn't create files)  
**Registry keys**: None  
**Services to check**: None  
**Firewall rules**: None  

## Pre-flight checks:
- Verify the target server has RSAT-AD-PowerShell and GPMC installed:
  ```
  - name: Check if required Windows features are installed
    ansible.windows.win_feature_info:
      name:
        - RSAT-AD-PowerShell
        - GPMC
    register: feature_status
  ```
- Verify connectivity to Active Directory:
  ```
  - name: Test Active Directory connectivity
    ansible.windows.win_shell: |
      Import-Module ActiveDirectory
      try {
        Get-ADDomain
        return $true
      } catch {
        return $false
      }
    register: ad_connectivity
  ```
- Verify the GPO backup source path exists and is accessible:
  ```
  - name: Check if GPO backup path exists
    ansible.windows.win_stat:
      path: "{{ gpo_backup_path }}"
    register: gpo_path_status
  ```