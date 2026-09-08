---
source-path: MemberServer.ps1
---

I now have all the information needed to write a comprehensive migration plan. Let me produce the full specification.

---

# Migration Plan: MemberServer.ps1

**TLDR**: This DSC Configuration defines a **baseline Windows Member Server** build for a high-security Enhanced Security Administrative Forest (ESAF). It is hosted on Azure Automation DSC and performs five core operations: enforcing UAC to "AlwaysNotify", setting the timezone to Pacific Standard Time, enforcing a PowerShell execution policy of `RemoteSigned`, creating an admin working directory, removing a sensitive API registration folder (to prevent key exposure after onboarding), joining the machine to an Active Directory domain, and installing the `BitLocker` Windows Feature. All variables and credentials are sourced from Azure Automation's secure vault at runtime.

---

## Service Type and Configuration

**Service Type**: Windows Member Server — Active Directory Domain Join / OS Baseline Hardening

**Key Operations**:
- Import DSC resource modules: `PSDesiredStateConfiguration`, `xPSDesiredStateConfiguration`, `ComputerManagementDSC`, `xSystemSecurity`, `xDSCDomainjoin`
- Retrieve runtime variables from Azure Automation (`ADMIN_PATH`, `API_FOLDER_PATH`, `DOMAIN_NAME`)
- Retrieve domain-join credential from Azure Automation Credential Store (`DOMAIN_JOIN`)
- Set UAC level to `AlwaysNotify` (most restrictive)
- Set system timezone to `Pacific Standard Time`
- Set PowerShell Execution Policy (scope: `LocalMachine`) to `RemoteSigned`
- Create admin directory at `$ADMIN_PATH` (e.g. `C:\Admin`)
- Delete (force-remove) the API registration folder at `$API_FOLDER_PATH` (e.g. `C:\Admin\One-Time-Config\DscMetaConfigs-Live`) to prevent API key exposure post-onboarding
- Join the machine to the Active Directory domain specified by `$DOMAIN_NAME`
- Install the `BitLocker` Windows Feature (including all sub-features)

---

## File Structure

**IMPORTANT: Files listed with relative paths from the repository root.**

**DSC Configurations:**
```
MemberServer.ps1
```

**Related Repository Files (context only — not part of this migration target):**
```
README.md
ActiveDirectoryBuild.ps1
ActiveDirectoryHub.ps1
AzureConnect.ps1
MemberServerSQL.ps1
RedForestBuild.ps1
S2DHypervisorDell.ps1
StandAloneHypervisorDell.ps1
runbooks/GenerateDSCConfig.ps1
runbooks/CaptureGuardedHost.ps1
runbooks/CaptureWindowsFeatures.ps1
runbooks/ConfigureSMBNICAdapters.ps1
runbooks/GenerateShadowPrinciples.ps1
runbooks/GetTrueUrl.ps1
runbooks/ReviewDangerousDirectoryChangesACL.ps1
```

> **Note**: Only `MemberServer.ps1` is in scope for this migration. The other files are listed for awareness only.

---

## Module Explanation

The configuration executes in the following order inside the `Node` block:

### 1. **Variable and Credential Initialization** (`MemberServer.ps1`, pre-Node block)

- `Get-AutomationVariable -Name "ADMIN_PATH"` — Retrieves the admin folder path (e.g. `C:\Admin`) from Azure Automation.
- `Get-AutomationVariable -Name "API_FOLDER_PATH"` — Retrieves the API registration folder path (e.g. `C:\Admin\One-Time-Config\DscMetaConfigs-Live`).
- `Get-AutomationVariable -Name "DOMAIN_NAME"` — Retrieves the Active Directory FQDN (e.g. `Cool-Name.Net`).
- `Get-AutomationPSCredential -Name "DOMAIN_JOIN"` — Retrieves the domain-join credential object (username + password) from the Azure Automation Credential Store.

**Ansible equivalent**: Use Ansible Vault or Azure Key Vault lookup to store and retrieve these values as Ansible variables (`admin_path`, `api_folder_path`, `domain_name`) and credentials (`domain_join_user`, `domain_join_password`).

---

### 2. **UAC Configuration** — DSC Resource: `xUAC` (`xSystemSecurity` module)

```powershell
xUAC UAC {
    Setting = "AlwaysNotify"
}
```

- Sets User Account Control to the most restrictive level: `AlwaysNotify`.
- This maps to registry key `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`:
  - `ConsentPromptBehaviorAdmin` = `2`
  - `ConsentPromptBehaviorUser` = `3`
  - `EnableLUA` = `1`
  - `PromptOnSecureDesktop` = `1`

**Ansible equivalent**: `ansible.windows.win_regedit` — set the four registry values listed above under `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`.

---

### 3. **Timezone Configuration** — DSC Resource: `TimeZone` (`ComputerManagementDSC` module)

```powershell
TimeZone TimeZoneSet {
    IsSingleInstance = 'Yes'
    TimeZone = 'Pacific Standard Time'
}
```

- Sets the system timezone to `Pacific Standard Time`.

**Ansible equivalent**: `community.windows.win_timezone`
```yaml
- name: Set timezone to Pacific Standard Time
  community.windows.win_timezone:
    timezone: "Pacific Standard Time"
```

---

### 4. **PowerShell Execution Policy** — DSC Resource: `PowerShellExecutionPolicy` (`ComputerManagementDSC` module)

```powershell
PowerShellExecutionPolicy PowerShellExecutionPolicySet {
    ExecutionPolicyScope = 'LocalMachine'
    ExecutionPolicy = 'RemoteSigned'
}
```

- Enforces `RemoteSigned` execution policy at the `LocalMachine` scope.

**Ansible equivalent**: `ansible.windows.win_shell` or `ansible.windows.win_regedit`
```yaml
- name: Set PowerShell execution policy to RemoteSigned
  ansible.windows.win_shell: |
    Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope LocalMachine -Force
```
Alternatively, set registry key `HKLM:\SOFTWARE\Microsoft\PowerShell\1\ShellIds\Microsoft.PowerShell` → `ExecutionPolicy` = `RemoteSigned`.

---

### 5. **Create Admin Directory** — DSC Resource: `File` (`PSDesiredStateConfiguration` module)

```powershell
File AdminFolder {
    Ensure = 'Present'
    Type = 'Directory'
    DestinationPath = $ADMIN_PATH   # e.g. C:\Admin
}
```

- Creates the admin working directory if it does not exist.

**Ansible equivalent**: `ansible.windows.win_file`
```yaml
- name: Create admin directory
  ansible.windows.win_file:
    path: "{{ admin_path }}"
    state: directory
```

---

### 6. **Remove API Registration Folder** — DSC Resource: `File` (`PSDesiredStateConfiguration` module)

```powershell
File RemoveAPIFolder {
    Ensure = 'Absent'
    Type = 'Directory'
    Force = $true
    DestinationPath = $API_FOLDER_PATH   # e.g. C:\Admin\One-Time-Config\DscMetaConfigs-Live
}
```

- **Force-deletes** the API registration folder and all its contents. This is a security measure to ensure the DSC onboarding API key is not left on disk after the node has registered with Azure Automation DSC.

**Ansible equivalent**: `ansible.windows.win_file`
```yaml
- name: Remove API registration folder (security cleanup)
  ansible.windows.win_file:
    path: "{{ api_folder_path }}"
    state: absent
```

---

### 7. **Domain Join** — DSC Resource: `xDSCDomainjoin` (`xDSCDomainjoin` module)

```powershell
xDSCDomainjoin JoinDomain {
    Domain = $DOMAIN_NAME
    Credential = $DOMAIN_JOIN
}
```

- Joins the machine to the Active Directory domain specified by `$DOMAIN_NAME` using the `$DOMAIN_JOIN` credential.
- This will trigger a **system reboot** upon successful domain join.

**Ansible equivalent**: `microsoft.ad.membership`
```yaml
- name: Join machine to Active Directory domain
  microsoft.ad.membership:
    dns_domain_name: "{{ domain_name }}"
    domain_admin_user: "{{ domain_join_user }}"
    domain_admin_password: "{{ domain_join_password }}"
    state: domain
    reboot: true
```

---

### 8. **Windows Feature Installation** — DSC Resource: `WindowsFeatureSet` (`PSDesiredStateConfiguration` module)

```powershell
$Features = @('BitLocker')

WindowsFeatureSet InstallFeatures {
    Name = $Features
    Ensure = 'Present'
    IncludeAllSubFeature = $true
}
```

- Installs the `BitLocker` Windows Feature along with all sub-features.

**Ansible equivalent**: `ansible.windows.win_feature`
```yaml
- name: Install BitLocker Windows Feature
  ansible.windows.win_feature:
    name: BitLocker
    state: present
    include_sub_features: true
```

> **Note**: BitLocker installation may require a reboot to complete. Use `register` + `ansible.windows.win_reboot` to handle this.

---

## PowerShell to Ansible Mapping

| PowerShell / DSC Operation | Ansible Module | Notes |
|---|---|---|
| `Get-AutomationVariable` | Ansible Vault / `vars` / Azure Key Vault lookup | Store `admin_path`, `api_folder_path`, `domain_name` as Ansible variables |
| `Get-AutomationPSCredential` | Ansible Vault | Store `domain_join_user` and `domain_join_password` in Ansible Vault |
| `xUAC` — `AlwaysNotify` | `ansible.windows.win_regedit` | Set 4 registry values under `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` |
| `TimeZone` — `Pacific Standard Time` | `community.windows.win_timezone` | `timezone: "Pacific Standard Time"` |
| `PowerShellExecutionPolicy` — `RemoteSigned` / `LocalMachine` | `ansible.windows.win_shell` or `ansible.windows.win_regedit` | `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope LocalMachine -Force` |
| `File` — `Ensure = 'Present'` / `Type = 'Directory'` | `ansible.windows.win_file` | `state: directory` |
| `File` — `Ensure = 'Absent'` / `Force = $true` | `ansible.windows.win_file` | `state: absent` — recursively removes directory |
| `xDSCDomainjoin` — domain join | `microsoft.ad.membership` | Triggers reboot; use `reboot: true` |
| `WindowsFeatureSet` — `BitLocker` / `IncludeAllSubFeature` | `ansible.windows.win_feature` | `include_sub_features: true`; may require reboot |

---

## Dependencies

**PowerShell DSC Module Dependencies**:
- `PSDesiredStateConfiguration` — built-in Windows DSC module (File, WindowsFeatureSet resources)
- `xPSDesiredStateConfiguration` — community DSC module (extended DSC resources)
- `ComputerManagementDSC` — community DSC module (TimeZone, PowerShellExecutionPolicy resources)
- `xSystemSecurity` — community DSC module (xUAC resource)
- `xDSCDomainjoin` — community DSC module (domain join resource)

**Ansible Collection Dependencies**:
- `ansible.windows` — core Windows modules (win_file, win_feature, win_regedit, win_shell, win_service)
- `community.windows` — community Windows modules (win_timezone)
- `microsoft.ad` — Active Directory modules (membership for domain join)

**Windows Features Installed**:
- `BitLocker` (with all sub-features)

**External Packages**: None

**Credentials Required**:
- `DOMAIN_JOIN` — Domain account with rights to join machines to AD (recommended: scoped to a specific OU only)

**Runtime Variables Required**:
| Variable Name | Example Value | Description |
|---|---|---|
| `ADMIN_PATH` | `C:\Admin` | Path for the admin working directory |
| `API_FOLDER_PATH` | `C:\Admin\One-Time-Config\DscMetaConfigs-Live` | Path to the DSC onboarding API key folder to be deleted post-registration |
| `DOMAIN_NAME` | `Cool-Name.Net` | Active Directory FQDN for domain join |

**Service Dependencies**: None explicitly managed (no `win_service` operations in this configuration)

**Reboot Triggers**:
- Domain join (`microsoft.ad.membership`) — always triggers reboot
- BitLocker feature installation — may require reboot

---

## Checks for the Migration

**Files to verify**:
- `{{ admin_path }}` (e.g. `C:\Admin`) — must exist after playbook run
- `{{ api_folder_path }}` (e.g. `C:\Admin\One-Time-Config\DscMetaConfigs-Live`) — must **not** exist after playbook run

**Registry keys to verify**:

| Registry Path | Value Name | Expected Value | Purpose |
|---|---|---|---|
| `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` | `ConsentPromptBehaviorAdmin` | `2` | UAC AlwaysNotify |
| `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` | `ConsentPromptBehaviorUser` | `3` | UAC AlwaysNotify |
| `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` | `EnableLUA` | `1` | UAC enabled |
| `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` | `PromptOnSecureDesktop` | `1` | UAC secure desktop |
| `HKLM:\SOFTWARE\Microsoft\PowerShell\1\ShellIds\Microsoft.PowerShell` | `ExecutionPolicy` | `RemoteSigned` | PS Execution Policy |

**Services to check**: None explicitly managed in this configuration.

**Windows Features to verify**:
- `BitLocker` — must be in `Installed` state

**Domain membership to verify**:
- Machine must be a member of `{{ domain_name }}`

**Timezone to verify**:
- System timezone must be `Pacific Standard Time`

---

## Pre-flight Checks

Before running the Ansible playbook, validate the following on the target node:

```powershell
# 1. Verify the target is NOT already domain-joined (or is joined to the correct domain)
(Get-WmiObject Win32_ComputerSystem).Domain

# 2. Verify network connectivity to the domain controller
Test-NetConnection -ComputerName $DOMAIN_NAME -Port 389

# 3. Verify the admin path parent directory exists and is writable
Test-Path (Split-Path $ADMIN_PATH -Parent)

# 4. Verify BitLocker feature availability (requires TPM or software encryption support)
Get-WindowsFeature -Name BitLocker

# 5. Verify current UAC setting
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" `
  -Name ConsentPromptBehaviorAdmin, ConsentPromptBehaviorUser, EnableLUA, PromptOnSecureDesktop

# 6. Verify current PowerShell execution policy
Get-ExecutionPolicy -Scope LocalMachine

# 7. Verify current timezone
Get-TimeZone

# 8. Confirm API folder exists BEFORE playbook run (so the cleanup task is meaningful)
Test-Path $API_FOLDER_PATH
```

**Post-run validation commands (run after Ansible playbook completes)**:

```powershell
# 1. Confirm domain membership
(Get-WmiObject Win32_ComputerSystem).Domain   # Should return DOMAIN_NAME

# 2. Confirm admin folder exists
Test-Path $ADMIN_PATH   # Should return True

# 3. Confirm API folder is gone
Test-Path $API_FOLDER_PATH   # Should return False

# 4. Confirm BitLocker is installed
(Get-WindowsFeature -Name BitLocker).InstallState   # Should return "Installed"

# 5. Confirm timezone
(Get-TimeZone).Id   # Should return "Pacific Standard Time"

# 6. Confirm execution policy
Get-ExecutionPolicy -Scope LocalMachine   # Should return "RemoteSigned"

# 7. Confirm UAC AlwaysNotify
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" `
  -Name ConsentPromptBehaviorAdmin   # Should return 2
```

---

## Important Migration Notes for the Junior Developer

1. **Azure Automation Variables → Ansible Variables**: The DSC config pulls `ADMIN_PATH`, `API_FOLDER_PATH`, and `DOMAIN_NAME` from Azure Automation at runtime. In Ansible, define these as playbook variables or group/host vars, and store sensitive values (credentials) in **Ansible Vault**.

2. **Reboot Handling**: Both the domain join and BitLocker installation may trigger reboots. Use `ansible.windows.win_reboot` after each task that may require it, or use the `reboot: true` parameter in `microsoft.ad.membership`. Structure your playbook so domain join happens **before** feature installation, and handle reboots between tasks.

3. **BitLocker Prerequisites**: BitLocker may require a TPM chip or a specific disk configuration. Ensure the target VM has a virtual TPM (vTPM) if running on Hyper-V Shielded VMs (as described in the README). The `win_feature` task will succeed even without TPM, but BitLocker **encryption** cannot be enabled without it.

4. **UAC Registry Values**: The `xUAC` DSC resource with `AlwaysNotify` maps to four specific registry DWORD values. Use four separate `ansible.windows.win_regedit` tasks (or one task with a loop) to set all four values.

5. **Domain Join Credential Scope**: Per the README, the `DOMAIN_JOIN` credential should only have rights to create computer objects in a **specific OU**. Ensure this is configured in Active Directory before running the playbook.

6. **Idempotency**: All Ansible modules listed are idempotent — running the playbook multiple times will not cause harm. The `win_file` with `state: absent` is safe to run even if the folder is already gone.

7. **Execution Order in Ansible**: Ansible executes tasks sequentially. Recommended order:
   - Set UAC → Set Timezone → Set Execution Policy → Create Admin Folder → Remove API Folder → Domain Join (+ reboot) → Install BitLocker (+ reboot if needed)