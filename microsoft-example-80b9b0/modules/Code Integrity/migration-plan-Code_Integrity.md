---
source-path: code-integrity
---

Now I'll create a migration plan for the Code Integrity module based on the XML files I've found:

# Migration Plan: Windows Code Integrity Configuration

**TLDR**: This module contains XML configuration files for Windows Code Integrity policies in both Audit and Enforce modes. These policies restrict which applications can run on Windows systems by allowing Microsoft-signed applications while denying specific bypass applications and PowerShell instances through file and hash rules.

## Service Type and Configuration

**Service Type**: Security (Windows Code Integrity)

**Key Operations**:
- Configures Windows Code Integrity policies in two modes: Audit and Enforce
- Blocks execution of potentially dangerous applications like debugging tools and PowerShell instances
- Allows specific Microsoft-signed applications and components
- Implements hash-based application allow/deny rules
- Enforces code signing requirements for applications

## File Structure

**XML Configuration Files:**
code-integrity/Audit/AllowMicrosoft_DenyByPassApps_Audit.xml
code-integrity/Enforce/AllowMicrosoft_DenyByPassApps_Enforce.xml

## Module Explanation

The Code Integrity module consists of XML configuration files that define Windows Defender Application Control (WDAC) policies. These policies control which applications can run on Windows systems.

1. **AllowMicrosoft_DenyByPassApps_Audit.xml** (`code-integrity/Audit/AllowMicrosoft_DenyByPassApps_Audit.xml`):
   - Configures Code Integrity in Audit mode (logs violations but doesn't block execution)
   - Allows Microsoft-signed applications
   - Denies specific applications that could bypass security (debugging tools, PowerShell instances)
   - Contains hash-based rules for blocking specific PowerShell instances
   - Ansible equivalent: win_dsc module with FileContentDsc resource or win_copy module

2. **AllowMicrosoft_DenyByPassApps_Enforce.xml** (`code-integrity/Enforce/AllowMicrosoft_DenyByPassApps_Enforce.xml`):
   - Similar to the Audit policy but in Enforce mode (blocks execution of non-compliant applications)
   - Contains the same deny rules but enforces them rather than just logging
   - Includes additional allow rules for specific Microsoft components
   - Ansible equivalent: win_dsc module with FileContentDsc resource or win_copy module

## PowerShell to Ansible Mapping

| PowerShell Operation | Ansible Module | Notes |
|---|---|---|
| Deploy Code Integrity XML files | ansible.windows.win_copy | Copy the XML files to the target location |
| Apply Code Integrity policy | community.windows.win_shell | Use `Set-CIPolicyIdInfo` and `ConvertFrom-CIPolicy` cmdlets |
| Enable Code Integrity | community.windows.win_shell | Use `CiTool.exe --update-policy` command |

## Dependencies

**PowerShell Module dependencies**: None explicitly defined
**Windows Features**: Windows Defender Application Control (built into Windows)
**External packages**: None
**Service dependencies**: None

## Checks for the Migration

**Files to verify**: 
- `C:\Windows\System32\CodeIntegrity\SIPolicy.p7b` (deployed policy)
- Event logs for Code Integrity events (Microsoft-Windows-CodeIntegrity/Operational)

## Pre-flight checks:
```powershell
# Check if Code Integrity is enabled
$ciStatus = Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\CI\Policy" -Name "VerifiedAndReputablePolicyState" -ErrorAction SilentlyContinue
if ($ciStatus -and $ciStatus.VerifiedAndReputablePolicyState -eq 1) {
    Write-Output "Code Integrity is enabled"
} else {
    Write-Output "Code Integrity is not enabled"
}

# Check for existing policies
if (Test-Path "C:\Windows\System32\CodeIntegrity\SIPolicy.p7b") {
    Write-Output "Code Integrity policy is deployed"
} else {
    Write-Output "No Code Integrity policy is deployed"
}
```

## Ansible Implementation Guide

### 1. Create Directory Structure

```yaml
- name: Ensure Code Integrity directories exist
  ansible.windows.win_file:
    path: "{{ item }}"
    state: directory
  loop:
    - "C:\\ConfigMgr\\CodeIntegrity\\Audit"
    - "C:\\ConfigMgr\\CodeIntegrity\\Enforce"
```

### 2. Deploy XML Policy Files

```yaml
- name: Copy Code Integrity Audit policy
  ansible.windows.win_copy:
    src: files/code-integrity/Audit/AllowMicrosoft_DenyByPassApps_Audit.xml
    dest: C:\ConfigMgr\CodeIntegrity\Audit\AllowMicrosoft_DenyByPassApps_Audit.xml

- name: Copy Code Integrity Enforce policy
  ansible.windows.win_copy:
    src: files/code-integrity/Enforce/AllowMicrosoft_DenyByPassApps_Enforce.xml
    dest: C:\ConfigMgr\CodeIntegrity\Enforce\AllowMicrosoft_DenyByPassApps_Enforce.xml
```

### 3. Apply Code Integrity Policy (Audit Mode)

```yaml
- name: Apply Code Integrity policy in Audit mode
  community.windows.win_shell: |
    $policyPath = "C:\ConfigMgr\CodeIntegrity\Audit\AllowMicrosoft_DenyByPassApps_Audit.xml"
    $outputPath = "C:\Windows\System32\CodeIntegrity\SIPolicy.p7b"
    
    # Convert the policy to binary format
    ConvertFrom-CIPolicy -XmlFilePath $policyPath -BinaryFilePath $outputPath
    
    # Apply the policy
    & CiTool.exe --update-policy $outputPath
  register: ci_result
  failed_when: "'successfully' not in ci_result.stdout"
```

### 4. Apply Code Integrity Policy (Enforce Mode)

```yaml
- name: Apply Code Integrity policy in Enforce mode
  community.windows.win_shell: |
    $policyPath = "C:\ConfigMgr\CodeIntegrity\Enforce\AllowMicrosoft_DenyByPassApps_Enforce.xml"
    $outputPath = "C:\Windows\System32\CodeIntegrity\SIPolicy.p7b"
    
    # Convert the policy to binary format
    ConvertFrom-CIPolicy -XmlFilePath $policyPath -BinaryFilePath $outputPath
    
    # Apply the policy
    & CiTool.exe --update-policy $outputPath
  register: ci_result
  failed_when: "'successfully' not in ci_result.stdout"
  when: enforce_mode | bool
```

### 5. Verify Code Integrity Status

```yaml
- name: Check Code Integrity status
  community.windows.win_shell: |
    $ciStatus = Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\CI\Policy" -Name "VerifiedAndReputablePolicyState" -ErrorAction SilentlyContinue
    if ($ciStatus -and $ciStatus.VerifiedAndReputablePolicyState -eq 1) {
        Write-Output "Code Integrity is enabled"
        exit 0
    } else {
        Write-Output "Code Integrity is not enabled"
        exit 1
    }
  register: ci_status_check
  failed_when: ci_status_check.rc != 0
  changed_when: false
```

### 6. Create a Role for Code Integrity

Create a dedicated Ansible role structure:

```
roles/
└── code_integrity/
    ├── defaults/
    │   └── main.yml
    ├── files/
    │   ├── Audit/
    │   │   └── AllowMicrosoft_DenyByPassApps_Audit.xml
    │   └── Enforce/
    │       └── AllowMicrosoft_DenyByPassApps_Enforce.xml
    ├── tasks/
    │   ├── main.yml
    │   ├── deploy_audit.yml
    │   └── deploy_enforce.yml
    └── handlers/
        └── main.yml
```

In `defaults/main.yml`:
```yaml
---
# Default to audit mode
enforce_mode: false
ci_base_path: "C:\\ConfigMgr\\CodeIntegrity"
```

In `tasks/main.yml`:
```yaml
---
- name: Include deploy tasks
  include_tasks: "{{ 'deploy_enforce.yml' if enforce_mode else 'deploy_audit.yml' }}"
```