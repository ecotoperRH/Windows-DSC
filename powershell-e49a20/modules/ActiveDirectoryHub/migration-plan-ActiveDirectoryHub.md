---
source-path: ActiveDirectoryHub.ps1
---

I have the full content of the file. Now let me provide the complete migration plan.

---

# Migration Plan: ActiveDirectoryHub.ps1

**TLDR**: This DSC Configuration provisions a Windows Server node as an **Active Directory Domain Controller** that joins an existing AD forest. It installs AD Domain Services, DNS, RSAT tools, and BitLocker features; enforces baseline OS settings (UAC, timezone, PowerShell execution policy); waits for the target forest to become available; promotes the node as a domain controller; and then monitors the NTDS service to ensure it remains running.

---

## Service Type and Configuration

**Service Type**: Active Directory Domain Controller (Hub / Replica DC joining an existing forest)

**Key Operations**:
- Import five DSC resource modules (`PSDesiredStateConfiguration`, `xPSDesiredStateConfiguration`, `ComputerManagementDSC`, `xSystemSecurity`, `ActiveDirectoryDsc`)
- Retrieve automation variables (`DOMAIN_NAME`, `ADMIN_PATH`, `API_FOLDER_PATH`) from Azure Automation
- Retrieve credentials (`DOMAIN_CONTROLLER_JOIN`, `DOMAIN_JOIN`) from Azure Automation credential store
- Set UAC to `AlwaysNotify`
- Set timezone to `Pacific Standard Time`
- Set PowerShell execution policy to `RemoteSigned` (scope: `LocalMachine`)
- Create an admin folder at `$ADMIN_PATH`
- Delete (force-remove) the API registration folder at `$API_FOLDER_PATH`
- Install Windows Features: `AD-Domain-Services`, `DNS`, `RSAT-AD-PowerShell`, `RSAT-ADDS`, `RSAT-DNS-Server`, `BitLocker`, `RSAT-Feature-Tools-BitLocker-BdeAducExt` (all with sub-features)
- Wait for the AD forest (`$DOMAIN_NAME`) to become available
- Promote the node as a Domain Controller and join the existing forest
- Monitor the `NTDS` service (ensure it is `Running` / `Automatic`)

---

## File Structure

**Scripts:**
```
ActiveDirectoryHub.ps1
```

**Modules:**
*(None — DSC modules are imported inline via `Import-DscResource`)*

**DSC Configurations:**
```
ActiveDirectoryHub.ps1
```

**Data Files:**
*(None — all variables are sourced from Azure Automation at runtime)*

---

## Module Explanation

The configuration executes in the following order:

### 1. **DSC Resource Imports** (`ActiveDirectoryHub.ps1`, top of Configuration block)
- `Import-DscResource -ModuleName 'PSDesiredStateConfiguration'` — provides core DSC resources (`File`, `Service`, `WindowsFeatureSet`)
- `Import-DscResource -ModuleName 'xPSDesiredStateConfiguration'` — provides extended DSC resources
- `Import-DscResource -ModuleName 'ComputerManagementDSC'` — provides `TimeZone`, `PowerShellExecutionPolicy`
- `Import-DscResource -ModuleName 'xSystemSecurity'` — provides `xUAC`
- `Import-DscResource -ModuleName 'ActiveDirectoryDsc'` — provides `WaitForADDomain`, `ADDomainController`
- **Ansible equivalent**: No direct equivalent; these are handled by the Ansible modules themselves. Ensure the target Windows host has the required DSC modules if any `win_dsc` tasks are used.

---

### 2. **Variable and Credential Retrieval** (`ActiveDirectoryHub.ps1`)
- `$DOMAIN_NAME = Get-AutomationVariable -Name "DOMAIN_NAME"` — the fully qualified domain name of the existing forest
- `$ADMIN_PATH = Get-AutomationVariable -Name "ADMIN_PATH"` — path for the admin folder to create
- `$API_FOLDER_PATH = Get-AutomationVariable -Name "API_FOLDER_PATH"` — path for the API folder to delete
- `$DOMAIN_CONTROLLER_JOIN = Get-AutomationPSCredential -Name 'DOMAIN_CONTROLLER_JOIN'` — credential used to promote the DC and set the DSRM password
- `$DOMAIN_JOIN = Get-AutomationPSCredential -Name 'DOMAIN_JOIN'` — credential used to wait/detect the forest
- **Ansible equivalent**: Define these as Ansible variables (`vars`, `group_vars`, or `vault`-encrypted secrets). Credentials map to `ansible_user`/`ansible_password` or dedicated vault variables.

---

### 3. **Base OS Settings** (`ActiveDirectoryHub.ps1` — Node block, first section)

#### 3a. UAC Configuration — `xUAC UAC`
- Sets UAC to `AlwaysNotify`
- **Ansible equivalent**: `ansible.windows.win_regedit` — set the registry key `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` values `ConsentPromptBehaviorAdmin = 2`, `ConsentPromptBehaviorUser = 3`, `PromptOnSecureDesktop = 1`

#### 3b. Timezone — `TimeZone TimeZoneSet`
- `IsSingleInstance = 'Yes'`, `TimeZone = 'Pacific Standard Time'`
- **Ansible equivalent**: `community.windows.win_timezone` with `timezone: "Pacific Standard Time"`

#### 3c. PowerShell Execution Policy — `PowerShellExecutionPolicy PowerShellExecutionPolicySet`
- `ExecutionPolicyScope = 'LocalMachine'`, `ExecutionPolicy = 'RemoteSigned'`
- **Ansible equivalent**: `ansible.windows.win_shell` running `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope LocalMachine -Force`, or `ansible.windows.win_regedit` targeting `HKLM:\SOFTWARE\Microsoft\PowerShell\1\ShellIds\Microsoft.PowerShell` key `ExecutionPolicy = RemoteSigned`

#### 3d. Create Admin Folder — `File AdminFolder`
- `Ensure = 'Present'`, `Type = 'Directory'`, `DestinationPath = $ADMIN_PATH`
- **Ansible equivalent**: `ansible.windows.win_file` with `path: "{{ admin_path }}"` and `state: directory`

#### 3e. Delete API Registration Folder — `File RemoveAPIFolder`
- `Ensure = 'Absent'`, `Type = 'Directory'`, `Force = $true`, `DestinationPath = $API_FOLDER_PATH`
- **Ansible equivalent**: `ansible.windows.win_file` with `path: "{{ api_folder_path }}"` and `state: absent`

---

### 4. **Windows Features Installation** (`ActiveDirectoryHub.ps1` — "Install Services" section)

#### `WindowsFeatureSet InstallFeatures`
- `Ensure = 'Present'`, `IncludeAllSubFeature = $true`
- Installs all 7 features in the `$Features` array:
  1. `AD-Domain-Services`
  2. `DNS`
  3. `RSAT-AD-PowerShell`
  4. `RSAT-ADDS`
  5. `RSAT-DNS-Server`
  6. `BitLocker`
  7. `RSAT-Feature-Tools-BitLocker-BdeAducExt`
- **Ansible equivalent**: `ansible.windows.win_feature` — one task with a `loop` over the feature names, or a single task using a list. Set `include_sub_features: true` and `include_management_tools: true`.

---

### 5. **Wait for AD Forest Availability** (`ActiveDirectoryHub.ps1`)

#### `WaitForADDomain WaitForestAvailability`
- `DomainName = $DOMAIN_NAME`
- `Credential = $DOMAIN_JOIN`
- `DependsOn = '[WindowsFeatureSet]InstallFeatures'`
- Polls until the specified domain/forest is reachable before proceeding
- **Ansible equivalent**: `ansible.windows.win_shell` with a retry loop using `until` / `retries` / `delay`:
  ```yaml
  - name: Wait for AD domain to be available
    ansible.windows.win_shell: |
      (Get-ADDomain -Identity "{{ domain_name }}" -Credential $cred).DNSRoot
    register: domain_check
    until: domain_check.rc == 0
    retries: 30
    delay: 30
  ```
  Alternatively use `ansible.windows.win_wait_for` if DNS resolution is sufficient.

---

### 6. **Promote Node as Domain Controller** (`ActiveDirectoryHub.ps1`)

#### `ADDomainController ForestJoin`
- `DomainName = $DOMAIN_NAME`
- `Credential = $DOMAIN_CONTROLLER_JOIN`
- `SafemodeAdministratorPassword = $DOMAIN_CONTROLLER_JOIN` (DSRM password)
- `DependsOn = '[WaitForADDomain]WaitForestAvailability'`
- Promotes the server as a replica DC in the existing forest
- **Ansible equivalent**: `microsoft.ad.domain_controller` (from the `microsoft.ad` collection) with:
  ```yaml
  - name: Promote server as Domain Controller
    microsoft.ad.domain_controller:
      dns_domain_name: "{{ domain_name }}"
      domain_admin_user: "{{ domain_controller_join_user }}"
      domain_admin_password: "{{ domain_controller_join_password }}"
      safe_mode_password: "{{ domain_controller_join_password }}"
      state: domain_controller
    register: dc_promotion
  ```
  A reboot will likely be required after promotion — use `ansible.windows.win_reboot`.

---

### 7. **Monitor NTDS Service** (`ActiveDirectoryHub.ps1` — "Monitor Services" section)

#### `Service NTDSService`
- `Name = 'NTDS'`
- `StartupType = 'Automatic'`
- `State = 'Running'`
- `DependsOn = '[ADDomainController]ForestJoin'`
- Ensures the Active Directory Domain Services (NTDS) service is running and set to auto-start
- **Ansible equivalent**: `ansible.windows.win_service` with:
  ```yaml
  - name: Ensure NTDS service is running
    ansible.windows.win_service:
      name: NTDS
      start_mode: auto
      state: started
  ```

---

## PowerShell to Ansible Mapping

| PowerShell / DSC Operation | Ansible Module | Notes |
|---|---|---|
| `Import-DscResource` (5 modules) | N/A | Handled natively by Ansible modules |
| `Get-AutomationVariable` | `vars` / `group_vars` / `ansible-vault` | Replace with Ansible variables or vault secrets |
| `Get-AutomationPSCredential` | `ansible-vault` encrypted vars | Store credentials as vault-encrypted variables |
| `xUAC` — `AlwaysNotify` | `ansible.windows.win_regedit` | Set `ConsentPromptBehaviorAdmin=2`, `ConsentPromptBehaviorUser=3`, `PromptOnSecureDesktop=1` under `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` |
| `TimeZone` — `Pacific Standard Time` | `community.windows.win_timezone` | `timezone: "Pacific Standard Time"` |
| `PowerShellExecutionPolicy` — `RemoteSigned` | `ansible.windows.win_shell` or `ansible.windows.win_regedit` | `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope LocalMachine -Force` |
| `File AdminFolder` (Directory, Present) | `ansible.windows.win_file` | `state: directory` |
| `File RemoveAPIFolder` (Directory, Absent) | `ansible.windows.win_file` | `state: absent` |
| `WindowsFeatureSet InstallFeatures` (7 features) | `ansible.windows.win_feature` | Loop over feature list; `include_sub_features: true` |
| `WaitForADDomain WaitForestAvailability` | `ansible.windows.win_shell` with `until/retries/delay` | Poll `Get-ADDomain` until reachable |
| `ADDomainController ForestJoin` | `microsoft.ad.domain_controller` | Requires `microsoft.ad` collection; triggers reboot |
| `Service NTDSService` (NTDS, Running, Automatic) | `ansible.windows.win_service` | `start_mode: auto`, `state: started` |

---

## Dependencies

**PowerShell DSC Module Dependencies**:
- `PSDesiredStateConfiguration` (built-in)
- `xPSDesiredStateConfiguration` (PowerShell Gallery)
- `ComputerManagementDSC` (PowerShell Gallery)
- `xSystemSecurity` (PowerShell Gallery)
- `ActiveDirectoryDsc` (PowerShell Gallery)

**Windows Features** (all with sub-features):
- `AD-Domain-Services`
- `DNS`
- `RSAT-AD-PowerShell`
- `RSAT-ADDS`
- `RSAT-DNS-Server`
- `BitLocker`
- `RSAT-Feature-Tools-BitLocker-BdeAducExt`

**External Packages**: None (no Chocolatey or Install-Package calls)

**Ansible Collection Dependencies**:
- `ansible.windows` (core Windows modules)
- `community.windows` (timezone, firewall, etc.)
- `microsoft.ad` (domain controller promotion)

**Service Dependencies**:
- `NTDS` (Active Directory Domain Services) — must be running after DC promotion

**Credential / Secret Dependencies** (from Azure Automation — must be migrated to Ansible Vault):
- `DOMAIN_CONTROLLER_JOIN` — used for DC promotion and DSRM password
- `DOMAIN_JOIN` — used to detect/wait for the forest

**Variable Dependencies** (from Azure Automation — must be migrated to Ansible vars):
- `DOMAIN_NAME` — FQDN of the existing AD forest/domain
- `ADMIN_PATH` — filesystem path for the admin folder
- `API_FOLDER_PATH` — filesystem path for the API folder to remove

---

## Checks for the Migration

**Files to verify**:
- `{{ admin_path }}` — must exist as a directory after playbook run
- `{{ api_folder_path }}` — must NOT exist after playbook run

**Registry keys**:
- `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`
  - `ConsentPromptBehaviorAdmin` = `2`
  - `ConsentPromptBehaviorUser` = `3`
  - `PromptOnSecureDesktop` = `1`
- `HKLM:\SOFTWARE\Microsoft\PowerShell\1\ShellIds\Microsoft.PowerShell`
  - `ExecutionPolicy` = `RemoteSigned`

**Services to check**:
- `NTDS` — State: Running, StartupType: Automatic
- `DNS` — State: Running (installed as part of DNS feature)

**Firewall rules**: None explicitly defined in this configuration.

---

## Pre-flight Checks

```yaml
# 1. Verify Windows Features are installed
- name: Check AD-Domain-Services feature
  ansible.windows.win_feature:
    name: AD-Domain-Services
    state: present
    include_sub_features: true

# 2. Verify timezone
- name: Verify timezone is Pacific Standard Time
  ansible.windows.win_shell: "(Get-TimeZone).Id"
  register: tz_check
  failed_when: "'Pacific Standard Time' not in tz_check.stdout"

# 3. Verify PowerShell execution policy
- name: Verify execution policy
  ansible.windows.win_shell: "Get-ExecutionPolicy -Scope LocalMachine"
  register: ep_check
  failed_when: "'RemoteSigned' not in ep_check.stdout"

# 4. Verify admin folder exists
- name: Check admin folder exists
  ansible.windows.win_stat:
    path: "{{ admin_path }}"
  register: admin_folder
  failed_when: not admin_folder.stat.exists

# 5. Verify API folder is absent
- name: Check API folder is removed
  ansible.windows.win_stat:
    path: "{{ api_folder_path }}"
  register: api_folder
  failed_when: api_folder.stat.exists

# 6. Verify NTDS service is running
- name: Check NTDS service state
  ansible.windows.win_service_info:
    name: NTDS
  register: ntds_svc
  failed_when: ntds_svc.services[0].state != 'started'

# 7. Verify domain controller is joined to the domain
- name: Verify DC is joined to domain
  ansible.windows.win_shell: "(Get-ADDomainController -Identity $env:COMPUTERNAME).Domain"
  register: dc_domain
  failed_when: domain_name not in dc_domain.stdout

# 8. Verify UAC registry settings
- name: Check UAC ConsentPromptBehaviorAdmin
  ansible.windows.win_regedit:
    path: HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System
    name: ConsentPromptBehaviorAdmin
    data: 2
    type: dword
    state: present
```

---

## Important Migration Notes for the Junior Developer

1. **Azure Automation Variables → Ansible Variables**: All `Get-AutomationVariable` calls must be replaced with Ansible variables defined in `group_vars`, `host_vars`, or passed via `--extra-vars`. Store sensitive values (credentials) in `ansible-vault`.

2. **Reboot after DC Promotion**: The `ADDomainController` DSC resource triggers an automatic reboot. In Ansible, you must explicitly handle this with `ansible.windows.win_reboot` after the `microsoft.ad.domain_controller` task.

3. **Dependency Chain**: The DSC `DependsOn` directives translate to Ansible task ordering. Ensure tasks are sequenced: Features → Wait for Domain → Promote DC → Verify NTDS service.

4. **`WaitForADDomain` Polling**: Implement a retry loop in Ansible (e.g., `retries: 30`, `delay: 30`) to replicate the polling behavior before attempting DC promotion.

5. **`microsoft.ad` Collection**: Install via `ansible-galaxy collection install microsoft.ad` before running the playbook.

6. **No Post-Install Services**: The original script explicitly notes "Do not add Post-Install Services to a HUB Domain Controller Build" — do not add extra service configuration tasks beyond what is listed above.