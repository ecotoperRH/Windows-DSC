---
source-path: code-integrity
---

Now that I've examined the files in the code-integrity directory, I'll create a migration plan for implementing these Code Integrity policies in Ansible.

# Migration Plan: Windows Code Integrity Policy

**TLDR**: This module contains Windows Code Integrity policy XML files that define security rules to restrict which applications can run on Windows systems. There are two policy files - one for audit mode and one for enforcement mode - that block potentially dangerous applications like debugging tools and PowerShell scripts with specific hash values, while allowing trusted Microsoft applications.

## Service Type and Configuration

**Service Type**: Security Configuration (Windows Defender Application Control)

**Key Operations**:
- Implements Windows Code Integrity policies to restrict which applications can run
- Provides two policy modes: Audit (logging only) and Enforce (blocking)
- Blocks potentially dangerous applications like debugging tools, PowerShell scripts, and other bypass techniques
- Allows only trusted Microsoft applications and specific whitelisted files
- Uses file hash values and certificate-based rules to identify allowed/denied applications

## File Structure

**XML Configuration Files:**
code-integrity/Audit/AllowMicrosoft_DenyByPassApps_Audit.xml
code-integrity/Enforce/AllowMicrosoft_DenyByPassApps_Enforce.xml

## Module Explanation

The code-integrity module contains XML policy files that define Windows Defender Application Control (WDAC) policies (formerly known as Device Guard Code Integrity policies). These policies determine which applications are allowed to run on Windows systems.

1. **AllowMicrosoft_DenyByPassApps_Audit.xml** (`code-integrity/Audit/AllowMicrosoft_DenyByPassApps_Audit.xml`):
   - Defines a Code Integrity policy in audit mode (logs violations but doesn't block)
   - Contains rules to allow Microsoft-signed applications
   - Denies specific applications that could be used to bypass security (debugging tools, PowerShell scripts)
   - Uses file name and hash-based rules to identify applications
   - Ansible equivalent: Use `win_copy` module to deploy the XML file to the target system

2. **AllowMicrosoft_DenyByPassApps_Enforce.xml** (`code-integrity/Enforce/AllowMicrosoft_DenyByPassApps_Enforce.xml`):
   - Similar to the audit policy but in enforcement mode (blocks non-compliant applications)
   - Contains the same allow/deny rules as the audit policy
   - Includes additional settings like "Required:Enforce Store Applications"
   - Ansible equivalent: Use `win_copy` module to deploy the XML file to the target system

After deploying the XML files, the policies need to be converted to binary format and applied to the system using PowerShell cmdlets.

## PowerShell to Ansible Mapping

| PowerShell Operation | Ansible Module | Notes |
|---|---|---|
| Copy-Item (XML files) | ansible.windows.win_copy | Copy the XML policy files to the target system |
| ConvertFrom-CIPolicy | ansible.windows.win_shell | Convert XML policy to binary format (.p7b or .cip) |
| Copy-Item (binary policy) | ansible.windows.win_copy | Copy the binary policy to the system policy location |
| Restart-Computer | ansible.windows.win_reboot | Restart to apply the policy (if needed) |

## Dependencies

**PowerShell Module dependencies**: ConfigCI module
**Windows Features**: Windows Defender Application Control (WDAC)
**External packages**: None
**Service dependencies**: None

## Checks for the Migration

**Files to verify**: 
- `C:\Windows\System32\CodeIntegrity\SIPolicy.p7b` (system-wide policy)
- `C:\Windows\System32\CodeIntegrity\CIPolicies\Active\{PolicyGUID}.cip` (if using multiple policies)

**Registry keys**: 
- `HKLM:\SYSTEM\CurrentControlSet\Control\CI\Policy`

**Services to check**: None specific, but verify Windows is booting correctly

## Implementation in Ansible

```yaml
---
- name: Deploy Windows Code Integrity Policies
  hosts: windows_servers
  gather_facts: yes
  vars:
    policy_mode: "enforce"  # Options: audit, enforce
    ci_policy_dir: "C:\\Windows\\System32\\CodeIntegrity"
  
  tasks:
    - name: Create Code Integrity directory if it doesn't exist
      ansible.windows.win_file:
        path: "{{ ci_policy_dir }}"
        state: directory
    
    - name: Copy Code Integrity policy XML file
      ansible.windows.win_copy:
        src: "code-integrity/{{ policy_mode | capitalize }}/AllowMicrosoft_DenyByPassApps_{{ policy_mode | capitalize }}.xml"
        dest: "{{ ci_policy_dir }}\\policy.xml"
    
    - name: Convert policy from XML to binary format
      ansible.windows.win_shell: |
        Import-Module ConfigCI
        ConvertFrom-CIPolicy -XmlFilePath "{{ ci_policy_dir }}\\policy.xml" -BinaryFilePath "{{ ci_policy_dir }}\\SIPolicy.p7b"
      register: convert_result
      failed_when: convert_result.rc != 0
    
    - name: Check if system needs reboot to apply policy
      ansible.windows.win_shell: |
        $currentPolicy = Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\CI\Policy" -ErrorAction SilentlyContinue
        if ($currentPolicy) {
          $policyGuid = [System.Guid]::Parse($currentPolicy.PolicyGuid)
          $newPolicyContent = Get-Content -Path "{{ ci_policy_dir }}\\SIPolicy.p7b" -Encoding Byte
          $newPolicyGuid = [System.Security.Cryptography.SHA256]::Create().ComputeHash($newPolicyContent) | ForEach-Object { $_.ToString("X2") }
          if ($policyGuid -ne $newPolicyGuid) { 
            Write-Output "REBOOT_REQUIRED"
          } else {
            Write-Output "NO_REBOOT_NEEDED"
          }
        } else {
          Write-Output "REBOOT_REQUIRED"
        }
      register: reboot_check
    
    - name: Reboot if needed
      ansible.windows.win_reboot:
        msg: "Rebooting to apply new Code Integrity policy"
        pre_reboot_delay: 15
        post_reboot_delay: 60
      when: reboot_check.stdout_lines[0] == "REBOOT_REQUIRED"
```

## Pre-flight checks:

```yaml
- name: Verify Code Integrity policy is applied
  ansible.windows.win_shell: |
    $ciPolicy = Get-CimInstance -ClassName Win32_SystemDriver -Filter "Name='CI'"
    if ($ciPolicy.State -eq 'Running') {
      $policyInfo = Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\CI\Policy" -ErrorAction SilentlyContinue
      if ($policyInfo) {
        Write-Output "Code Integrity policy is active. Policy GUID: $($policyInfo.PolicyGuid)"
        
        # Check if in audit mode
        $auditMode = $policyInfo.Options -band 0x1
        if ($auditMode -eq 0x1) {
          Write-Output "Policy is in AUDIT mode"
        } else {
          Write-Output "Policy is in ENFORCE mode"
        }
        
        exit 0
      } else {
        Write-Output "No Code Integrity policy found"
        exit 1
      }
    } else {
      Write-Output "Code Integrity service is not running"
      exit 2
    }
  register: ci_check
  failed_when: ci_check.rc != 0
```