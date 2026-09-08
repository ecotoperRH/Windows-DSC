---
source-path: AzureConnect.ps1
---

I now have the full content of the file. Let me write the detailed migration plan.

---

# Migration Plan: AzureConnect.ps1

**TLDR**: This is a PowerShell DSC (Desired State Configuration) script that performs baseline OS hardening and domain-join operations on a Windows node provisioned through Azure Automation. It configures UAC, timezone, PowerShell execution policy, creates an admin directory, removes an API registration folder, joins the machine to an Active Directory domain, and installs the BitLocker Windows feature. Credentials and path variables are sourced from Azure Automation's secure variable/credential store.

---

## Service Type and Configuration

**Service Type**: Base OS Hardening / Domain Join / Azure Automation DSC

**Key Operations**:
- Retrieve automation variables (`ADMIN_PATH`, `API_FOLDER_PATH`, `DOMAIN_NAME`) from Azure Automation
- Retrieve domain-join credential (`DOMAIN_JOIN`) from Azure Automation credential store
- Set UAC to `AlwaysNotify`
- Set timezone to `Pacific Standard Time`
- Set PowerShell Execution Policy to `RemoteSigned` (scope: `LocalMachine`)
- Create the admin folder at `$ADMIN_PATH` (ensure present)
- Delete the API registration folder at `$API_FOLDER_PATH` (ensure absent, forced)
- Join the machine to the Active Directory domain `$DOMAIN_NAME` using `$DOMAIN_JOIN` credentials
- Install Windows Feature: `BitLocker` (including all sub-features)

---

## File Structure

**Scripts / DSC Configurations:**
```
AzureConnect.ps1
```

**Data Files:**
- None (variables and credentials are sourced at runtime from Azure Automation, not from `.psd1` files)

---

## Module Explanation

The script performs operations in this order:

### 1. **AzureConnect DSC Configuration** (`AzureConnect.ps1`)

#### 1a. Import DSC Resource Modules
The configuration imports the following DSC resource modules:
- `PSDesiredStateConfiguration` — built-in DSC resources (`File`, `WindowsFeatureSet`)
- `xPSDesiredStateConfiguration` — extended DSC resources
- `ComputerManagementDSC` — provides `TimeZone` and `PowerShellExecutionPolicy` resources
- `xSystemSecurity` — provides `xUAC` resource
- `xDSCDomainjoin` — provides `xDSCDomainjoin` resource for AD domain join

**Ansible equivalent**: No direct action needed; Ansible modules replace these DSC resources natively.

---

#### 1b. Retrieve Azure Automation Variables and Credentials
```powershell
$ADMIN_PATH    = Get-AutomationVariable -Name "ADMIN_PATH"
$API_FOLDER_PATH = Get-AutomationVariable -Name "API_FOLDER_PATH"
$DOMAIN_NAME   = Get-AutomationVariable -Name "DOMAIN_NAME"
$DOMAIN_JOIN   = Get-AutomationPSCredential -Name "DOMAIN_JOIN"
```
These are runtime values injected by Azure Automation. In Ansible, these become **inventory variables** or **Ansible Vault secrets** passed as `vars` or `group_vars`.

**Ansible equivalent**: Define `admin_path`, `api_folder_path`, `domain_name`, `domain_join_user`, and `domain_join_password` as Ansible variables (preferably stored in Ansible Vault for credentials).

---

#### 1c. Define Windows Features List
```powershell
$Features = @('BitLocker')
```
A single-element array used by `WindowsFeatureSet`.

**Ansible equivalent**: A YAML list variable `windows_features: ['BitLocker']` iterated with `ansible.windows.win_feature`.

---

#### 1d. DSC Resource: `xUAC` — Set UAC to AlwaysNotify
```powershell
xUAC UAC {
    Setting = "AlwaysNotify"
}
```
Sets User Account Control to the most restrictive setting (`AlwaysNotify`), which corresponds to registry key `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`.

**Ansible equivalent**: `ansible.windows.win_regedit`
- Key: `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`
- Values to set:
  - `EnableLUA` = `1` (DWORD)
  - `ConsentPromptBehaviorAdmin` = `2` (DWORD)
  - `ConsentPromptBehaviorUser` = `3` (DWORD)
  - `PromptOnSecureDesktop` = `1` (DWORD)

---

#### 1e. DSC Resource: `TimeZone` — Set Timezone
```powershell
TimeZone TimeZoneSet {
    IsSingleInstance = 'Yes'
    TimeZone = 'Pacific Standard Time'
}
```
Sets the system timezone to Pacific Standard Time.

**Ansible equivalent**: `community.windows.win_timezone`
- `timezone: "Pacific Standard Time"`

---

#### 1f. DSC Resource: `PowerShellExecutionPolicy` — Set Execution Policy
```powershell
PowerShellExecutionPolicy PowerShellExecutionPolicySet {
    ExecutionPolicyScope = 'LocalMachine'
    ExecutionPolicy = 'RemoteSigned'
}
```
Sets the PowerShell execution policy for the `LocalMachine` scope to `RemoteSigned`.

**Ansible equivalent**: `ansible.windows.win_shell`
```yaml
ansible.windows.win_shell:
  _raw_params: Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope LocalMachine -Force
```
Or via registry:
- Key: `HKLM:\SOFTWARE\Microsoft\PowerShell\1\ShellIds\Microsoft.PowerShell`
- Value: `ExecutionPolicy` = `RemoteSigned` (String)

**Ansible equivalent**: `ansible.windows.win_regedit`

---

#### 1g. DSC Resource: `File AdminFolder` — Create Admin Directory
```powershell
File AdminFolder {
    Ensure = 'Present'
    Type = 'Directory'
    DestinationPath = $ADMIN_PATH
}
```
Creates the directory at the path stored in `$ADMIN_PATH` if it does not already exist.

**Ansible equivalent**: `ansible.windows.win_file`
```yaml
ansible.windows.win_file:
  path: "{{ admin_path }}"
  state: directory
```

---

#### 1h. DSC Resource: `File RemoveAPIFolder` — Delete API Registration Folder
```powershell
File RemoveAPIFolder {
    Ensure = 'Absent'
    Type = 'Directory'
    Force = $true
    DestinationPath = $API_FOLDER_PATH
}
```
Forcefully removes the directory at `$API_FOLDER_PATH` and all its contents if it exists.

**Ansible equivalent**: `ansible.windows.win_file`
```yaml
ansible.windows.win_file:
  path: "{{ api_folder_path }}"
  state: absent
```

---

#### 1i. DSC Resource: `xDSCDomainjoin JoinDomain` — Join Active Directory Domain
```powershell
xDSCDomainjoin JoinDomain {
    Domain = $DOMAIN_NAME
    Credential = $DOMAIN_JOIN
}
```
Joins the Windows node to the Active Directory domain specified by `$DOMAIN_NAME` using the credentials in `$DOMAIN_JOIN`.

**Ansible equivalent**: `microsoft.ad.membership` (preferred) or `ansible.windows.win_domain_membership`
```yaml
microsoft.ad.membership:
  dns_domain_name: "{{ domain_name }}"
  domain_admin_user: "{{ domain_join_user }}"
  domain_admin_password: "{{ domain_join_password }}"
  state: domain
  reboot: true
```

---

#### 1j. DSC Resource: `WindowsFeatureSet InstallFeatures` — Install BitLocker
```powershell
WindowsFeatureSet InstallFeatures {
    Name = $Features   # ['BitLocker']
    Ensure = 'Present'
    IncludeAllSubFeature = $true
}
```
Installs the `BitLocker` Windows feature along with all its sub-features.

**Ansible equivalent**: `ansible.windows.win_feature`
```yaml
ansible.windows.win_feature:
  name: BitLocker
  state: present
  include_sub_features: true
```

---

## PowerShell to Ansible Mapping

| PowerShell / DSC Operation | Ansible Module | Notes |
|---|---|---|
| `Get-AutomationVariable` | Ansible `vars` / `group_vars` / Ansible Vault | Replace Azure Automation variables with Ansible variables |
| `Get-AutomationPSCredential` | Ansible Vault secrets | Store `domain_join_user` and `domain_join_password` in Vault |
| `xUAC` — `AlwaysNotify` | `ansible.windows.win_regedit` | Set `EnableLUA`, `ConsentPromptBehaviorAdmin`, `ConsentPromptBehaviorUser`, `PromptOnSecureDesktop` registry values |
| `TimeZone` — `Pacific Standard Time` | `community.windows.win_timezone` | `timezone: "Pacific Standard Time"` |
| `PowerShellExecutionPolicy` — `RemoteSigned` / `LocalMachine` | `ansible.windows.win_regedit` or `ansible.windows.win_shell` | Set via registry or `Set-ExecutionPolicy` shell command |
| `File` (Ensure = Present, Directory) | `ansible.windows.win_file` | `state: directory` |
| `File` (Ensure = Absent, Force) | `ansible.windows.win_file` | `state: absent` |
| `xDSCDomainjoin` | `microsoft.ad.membership` | Requires reboot after domain join |
| `WindowsFeatureSet` — `BitLocker` | `ansible.windows.win_feature` | `include_sub_features: true` |

---

## Dependencies

**PowerShell DSC Module Dependencies**:
- `PSDesiredStateConfiguration` (built-in)
- `xPSDesiredStateConfiguration`
- `ComputerManagementDSC`
- `xSystemSecurity`
- `xDSCDomainjoin`

**Windows Features**:
- `BitLocker` (with all sub-features)

**External Packages**:
- None

**Azure Automation Dependencies** (must be migrated to Ansible variables/vault):
- Variable: `ADMIN_PATH` → `{{ admin_path }}`
- Variable: `API_FOLDER_PATH` → `{{ api_folder_path }}`
- Variable: `DOMAIN_NAME` → `{{ domain_name }}`
- Credential: `DOMAIN_JOIN` → `{{ domain_join_user }}` / `{{ domain_join_password }}`

**Service Dependencies**:
- Active Directory domain controller must be reachable at time of domain join
- DNS must resolve `$DOMAIN_NAME` from the target node

---

## Checks for the Migration

**Files to verify**:
- `{{ admin_path }}` — must exist as a directory after playbook run
- `{{ api_folder_path }}` — must NOT exist after playbook run

**Registry keys**:
- `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`
  - `EnableLUA` = `1`
  - `ConsentPromptBehaviorAdmin` = `2`
  - `ConsentPromptBehaviorUser` = `3`
  - `PromptOnSecureDesktop` = `1`
- `HKLM:\SOFTWARE\Microsoft\PowerShell\1\ShellIds\Microsoft.PowerShell`
  - `ExecutionPolicy` = `RemoteSigned`

**Services to check**:
- `BitLocker Drive Encryption Service` (`BDESVC`) — should be present after feature install

**Windows Features to check**:
- `BitLocker` — installed and all sub-features present

**Domain membership**:
- Node must be a member of `{{ domain_name }}` after playbook run

**Timezone**:
- System timezone must be `Pacific Standard Time`

---

## Pre-flight Checks

Before running the Ansible playbook, validate the following:

```yaml
# 1. Verify DNS resolution of the domain from the target node
- name: Check domain DNS resolution
  ansible.windows.win_shell: Resolve-DnsName "{{ domain_name }}"
  register: dns_check

# 2. Verify domain controller is reachable (port 389 LDAP)
- name: Check LDAP port to domain controller
  ansible.windows.win_shell: Test-NetConnection -ComputerName "{{ domain_name }}" -Port 389
  register: ldap_check

# 3. Verify domain join credentials are valid before attempting join
- name: Validate domain join credential variable is defined
  ansible.builtin.assert:
    that:
      - domain_join_user is defined
      - domain_join_password is defined
      - domain_name is defined
      - admin_path is defined
      - api_folder_path is defined

# 4. Verify BitLocker feature availability (not available on all SKUs)
- name: Check BitLocker feature availability
  ansible.windows.win_shell: Get-WindowsFeature -Name BitLocker
  register: bitlocker_check

# 5. Post-run: Verify domain membership
- name: Verify domain membership
  ansible.windows.win_shell: (Get-WmiObject Win32_ComputerSystem).Domain
  register: domain_result

# 6. Post-run: Verify admin folder exists
- name: Verify admin folder exists
  ansible.windows.win_stat:
    path: "{{ admin_path }}"
  register: admin_folder_stat

# 7. Post-run: Verify API folder is absent
- name: Verify API folder is absent
  ansible.windows.win_stat:
    path: "{{ api_folder_path }}"
  register: api_folder_stat
```

---

## Important Migration Notes for the Junior Developer

1. **Azure Automation Variables → Ansible Variables**: The DSC script pulls `ADMIN_PATH`, `API_FOLDER_PATH`, and `DOMAIN_NAME` from Azure Automation's variable store at runtime. In Ansible, define these as playbook variables, `group_vars`, or `host_vars`. Use **Ansible Vault** for the domain join credentials.

2. **Reboot after Domain Join**: The `xDSCDomainjoin` resource triggers a reboot automatically. In Ansible, set `reboot: true` in the `microsoft.ad.membership` task, or use a separate `ansible.windows.win_reboot` task immediately after.

3. **UAC Registry Values**: The `xUAC` DSC resource with `AlwaysNotify` maps to four specific registry DWORD values. All four must be set together to achieve the correct UAC behavior.

4. **BitLocker Sub-features**: `IncludeAllSubFeature = $true` in DSC maps to `include_sub_features: true` in `ansible.windows.win_feature`. This is important — omitting it may result in an incomplete BitLocker installation.

5. **Force Delete of API Folder**: The `Force = $true` on the `File RemoveAPIFolder` resource means it will recursively delete the folder and all contents. `ansible.windows.win_file` with `state: absent` handles this recursively by default on directories.

6. **Idempotency**: All Ansible modules listed are idempotent — re-running the playbook will not cause unintended changes if the desired state is already met.