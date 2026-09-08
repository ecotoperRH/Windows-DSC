---
source-path: S2DHypervisorDell.ps1
---

I now have the full content of the file. Let me analyze it thoroughly and produce the migration plan.

---

# Migration Plan: S2DHypervisorDell.ps1

**TLDR**: This PowerShell DSC Configuration script provisions a **Storage Spaces Direct (S2D) Hyper-V hypervisor node** on Dell hardware. It configures base OS settings (UAC, timezone, PowerShell execution policy), manages administrative directories, joins the node to an Active Directory domain, installs a comprehensive set of Windows Features (Hyper-V, BitLocker, Host Guardian, Failover Clustering, etc.), and creates an external SET (Switch Embedded Teaming) virtual switch named `OS-Traffic` across two NICs. Credentials and path variables are sourced from Azure Automation.

---

## Service Type and Configuration

**Service Type**: Hyper-V Hypervisor / Storage Spaces Direct (S2D) Cluster Node

**Key Operations**:
- Set UAC to `AlwaysNotify`
- Set timezone to `Pacific Standard Time`
- Set PowerShell Execution Policy to `RemoteSigned` (LocalMachine scope)
- Create an admin directory at `$ADMIN_PATH`
- Delete/remove an API registration directory at `$API_FOLDER_PATH`
- Join the node to an Active Directory domain (`$DOMAIN_NAME`)
- Install 13 Windows Features (with all sub-features): `Hyper-V`, `BitLocker`, `HostGuardian`, `RSAT-Shielded-VM-Tools`, `RSAT-Hyper-V-Tools`, `Hyper-V-Tools`, `Hyper-V-PowerShell`, `Failover-Clustering`, `RSAT-Clustering-PowerShell`, `FS-FileServer`, `NetworkVirtualization`, `RSAT-Clustering-Mgmt`, `Data-Center-Bridging`
- Create an external Hyper-V virtual switch named `OS-Traffic` using Switch Embedded Teaming (SET) across `NIC1` and `NIC2`, with management OS access enabled

---

## File Structure

**Scripts:**
```
S2DHypervisorDell.ps1
```

**Modules:**
*(None — DSC modules are imported inline via `Import-DscResource`)*

**DSC Configurations:**
```
S2DHypervisorDell.ps1
```

**Data Files:**
*(None — variables are sourced from Azure Automation at runtime)*

---

## Module Explanation

The script contains a single DSC `Configuration` block named `S2DHypervisorDell`. All operations are declared inside a single `Node` block. Execution order follows DSC dependency resolution (explicit `DependsOn` and implicit ordering).

### 1. **Variable & Credential Initialization** (`S2DHypervisorDell.ps1`, top of Configuration block)

- `$ADMIN_PATH` — Retrieved from Azure Automation (`Get-AutomationVariable`). Represents the path for the admin folder.
- `$API_FOLDER_PATH` — Retrieved from Azure Automation. Represents the path for the API registration folder to be removed.
- `$DOMAIN_NAME` — Retrieved from Azure Automation. The Active Directory domain to join.
- `$DOMAIN_JOIN` — Retrieved from Azure Automation credential store (`Get-AutomationPSCredential`). Used for domain join authentication.
- `$Features` — A hardcoded array of 13 Windows Feature names to install.

**Ansible equivalent**: Define these as Ansible variables in `group_vars`, `host_vars`, or a vault-encrypted file. Credentials should be stored in Ansible Vault.

---

### 2. **UAC Configuration** — DSC Resource: `xUAC`

- **DSC Resource**: `xUAC` (from `xSystemSecurity` module)
- **Setting**: `AlwaysNotify`
- **What it does**: Sets User Account Control to the most restrictive level — always prompt for both app installs and Windows settings changes.

**Ansible equivalent**: `ansible.windows.win_regedit`
- Registry path: `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`
- Keys to set:
  - `ConsentPromptBehaviorAdmin` = `2` (Prompt for credentials on secure desktop)
  - `ConsentPromptBehaviorUser` = `3`
  - `EnableLUA` = `1`
  - `PromptOnSecureDesktop` = `1`

---

### 3. **Timezone Configuration** — DSC Resource: `TimeZone`

- **DSC Resource**: `TimeZone` (from `ComputerManagementDSC` module)
- **IsSingleInstance**: `Yes`
- **TimeZone**: `Pacific Standard Time`

**Ansible equivalent**: `community.windows.win_timezone`
```yaml
- name: Set timezone to Pacific Standard Time
  community.windows.win_timezone:
    timezone: "Pacific Standard Time"
```

---

### 4. **PowerShell Execution Policy** — DSC Resource: `PowerShellExecutionPolicy`

- **DSC Resource**: `PowerShellExecutionPolicy` (from `ComputerManagementDSC` module)
- **ExecutionPolicyScope**: `LocalMachine`
- **ExecutionPolicy**: `RemoteSigned`

**Ansible equivalent**: `ansible.windows.win_shell`
```yaml
- name: Set PowerShell execution policy
  ansible.windows.win_shell: |
    Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope LocalMachine -Force
```
Or via registry:
- `HKLM:\SOFTWARE\Microsoft\PowerShell\1\ShellIds\Microsoft.PowerShell` → `ExecutionPolicy` = `RemoteSigned`

---

### 5. **Create Admin Folder** — DSC Resource: `File AdminFolder`

- **DSC Resource**: `File` (from `PSDesiredStateConfiguration`)
- **Ensure**: `Present`
- **Type**: `Directory`
- **DestinationPath**: `$ADMIN_PATH` (value from Azure Automation variable)

**Ansible equivalent**: `ansible.windows.win_file`
```yaml
- name: Create admin folder
  ansible.windows.win_file:
    path: "{{ admin_path }}"
    state: directory
```

---

### 6. **Remove API Registration Folder** — DSC Resource: `File RemoveAPIFolder`

- **DSC Resource**: `File` (from `PSDesiredStateConfiguration`)
- **Ensure**: `Absent`
- **Type**: `Directory`
- **Force**: `$true` (recursive delete)
- **DestinationPath**: `$API_FOLDER_PATH` (value from Azure Automation variable)

**Ansible equivalent**: `ansible.windows.win_file`
```yaml
- name: Remove API registration folder
  ansible.windows.win_file:
    path: "{{ api_folder_path }}"
    state: absent
```

---

### 7. **Domain Join** — DSC Resource: `xDSCDomainjoin`

- **DSC Resource**: `xDSCDomainjoin` (from `xDSCDomainjoin` module)
- **Domain**: `$DOMAIN_NAME`
- **Credential**: `$DOMAIN_JOIN` (PSCredential from Azure Automation vault)

**Ansible equivalent**: `microsoft.ad.membership` (or `ansible.windows.win_domain_membership`)
```yaml
- name: Join Active Directory domain
  microsoft.ad.membership:
    dns_domain_name: "{{ domain_name }}"
    domain_admin_user: "{{ domain_join_user }}"
    domain_admin_password: "{{ domain_join_password }}"
    state: domain
    reboot: true
```

---

### 8. **Windows Features Installation** — DSC Resource: `WindowsFeatureSet InstallFeatures`

- **DSC Resource**: `WindowsFeatureSet` (from `PSDesiredStateConfiguration`)
- **Ensure**: `Present`
- **IncludeAllSubFeature**: `$true`
- **Features installed** (all 13, explicitly):

| # | Feature Name | Purpose |
|---|---|---|
| 1 | `Hyper-V` | Core hypervisor role |
| 2 | `BitLocker` | Drive encryption |
| 3 | `HostGuardian` | Guarded Fabric / Shielded VMs host service |
| 4 | `RSAT-Shielded-VM-Tools` | Remote tools for shielded VM management |
| 5 | `RSAT-Hyper-V-Tools` | Remote tools for Hyper-V management |
| 6 | `Hyper-V-Tools` | Hyper-V GUI management tools |
| 7 | `Hyper-V-PowerShell` | Hyper-V PowerShell module |
| 8 | `Failover-Clustering` | Windows Server Failover Clustering (for S2D) |
| 9 | `RSAT-Clustering-PowerShell` | PowerShell tools for cluster management |
| 10 | `FS-FileServer` | File Server role (required for S2D) |
| 11 | `NetworkVirtualization` | Hyper-V Network Virtualization |
| 12 | `RSAT-Clustering-Mgmt` | Failover Cluster Manager GUI |
| 13 | `Data-Center-Bridging` | DCB/QoS for RDMA networking (critical for S2D) |

**Ansible equivalent**: `ansible.windows.win_feature` — one task per feature, or use a loop:
```yaml
- name: Install Windows Features for S2D Hyper-V
  ansible.windows.win_feature:
    name: "{{ item }}"
    state: present
    include_sub_features: true
    include_management_tools: true
  loop:
    - Hyper-V
    - BitLocker
    - HostGuardian
    - RSAT-Shielded-VM-Tools
    - RSAT-Hyper-V-Tools
    - Hyper-V-Tools
    - Hyper-V-PowerShell
    - Failover-Clustering
    - RSAT-Clustering-PowerShell
    - FS-FileServer
    - NetworkVirtualization
    - RSAT-Clustering-Mgmt
    - Data-Center-Bridging
  register: feature_install
```
> ⚠️ **Note**: A reboot is likely required after installing `Hyper-V` and `HostGuardian`. Add a `win_reboot` task after this step.

---

### 9. **Hyper-V External Virtual Switch (SET)** — DSC Resource: `xVMSwitch ExternalVSwitch`

- **DSC Resource**: `xVMSwitch` (from `xHyper-V` module)
- **Name**: `OS-Traffic`
- **Type**: `External`
- **AllowManagementOS**: `$true` (host OS shares the switch)
- **Ensure**: `Present`
- **NetAdapterName**: `NIC1`, `NIC2` (two physical NICs teamed via SET)
- **EnableEmbeddedTeaming**: `$true` (Switch Embedded Teaming — Hyper-V native NIC teaming)
- **DependsOn**: `[WindowsFeatureSet]InstallFeatures` (Hyper-V must be installed first)

**Ansible equivalent**: `ansible.windows.win_shell` (no native Ansible module for SET vSwitch creation)
```yaml
- name: Create External SET vSwitch OS-Traffic
  ansible.windows.win_shell: |
    $switch = Get-VMSwitch -Name 'OS-Traffic' -ErrorAction SilentlyContinue
    if (-not $switch) {
      New-VMSwitch -Name 'OS-Traffic' `
        -NetAdapterName 'NIC1','NIC2' `
        -EnableEmbeddedTeaming $true `
        -AllowManagementOS $true
    }
  args:
    executable: powershell.exe
```
> ⚠️ **Note**: This task must run **after** the Hyper-V feature is installed and the node has been rebooted.

---

## PowerShell to Ansible Mapping

| PowerShell / DSC Operation | Ansible Module | Notes |
|---|---|---|
| `Get-AutomationVariable` | Ansible `vars` / `group_vars` / `host_vars` | Replace Azure Automation variables with Ansible variables |
| `Get-AutomationPSCredential` | `ansible.builtin.vault` / Ansible Vault | Store domain join credentials in Ansible Vault |
| `xUAC` (Setting = AlwaysNotify) | `ansible.windows.win_regedit` | Set UAC registry keys under `HKLM:\...\Policies\System` |
| `TimeZone` (Pacific Standard Time) | `community.windows.win_timezone` | Direct module mapping |
| `PowerShellExecutionPolicy` (RemoteSigned) | `ansible.windows.win_shell` | `Set-ExecutionPolicy` or `win_regedit` |
| `File` (Ensure = Present, Directory) | `ansible.windows.win_file` | `state: directory` |
| `File` (Ensure = Absent, Directory, Force) | `ansible.windows.win_file` | `state: absent` |
| `xDSCDomainjoin` | `microsoft.ad.membership` | Requires reboot after join |
| `WindowsFeatureSet` (13 features) | `ansible.windows.win_feature` | Loop over feature list; `include_sub_features: true` |
| `xVMSwitch` (External, SET, 2 NICs) | `ansible.windows.win_shell` | No native module; use `New-VMSwitch` via shell |

---

## Dependencies

**PowerShell DSC Module Dependencies**:
- `PSDesiredStateConfiguration` — Built-in DSC resources (`File`, `WindowsFeatureSet`)
- `xPSDesiredStateConfiguration` — Extended DSC resources
- `xHyper-V` — `xVMSwitch` resource
- `ComputerManagementDSC` — `TimeZone`, `PowerShellExecutionPolicy` resources
- `xSystemSecurity` — `xUAC` resource
- `xDSCDomainjoin` — Domain join resource
- `GuardedFabricTools` — Guarded Fabric / Host Guardian tooling (imported but no explicit resource used in this script)

**Windows Features** (all installed with sub-features):
- `Hyper-V`, `BitLocker`, `HostGuardian`, `RSAT-Shielded-VM-Tools`, `RSAT-Hyper-V-Tools`, `Hyper-V-Tools`, `Hyper-V-PowerShell`, `Failover-Clustering`, `RSAT-Clustering-PowerShell`, `FS-FileServer`, `NetworkVirtualization`, `RSAT-Clustering-Mgmt`, `Data-Center-Bridging`

**External Packages**: None (no Chocolatey or Install-Package calls)

**Service Dependencies**:
- Hyper-V Virtual Machine Management (`vmms`) — started by Hyper-V feature install
- Host Guardian Service (`hgs`) — started by HostGuardian feature
- Cluster Service (`ClusSvc`) — started by Failover-Clustering feature

**Runtime Variable Sources** (Azure Automation — must be migrated to Ansible):
- `ADMIN_PATH` → `{{ admin_path }}`
- `API_FOLDER_PATH` → `{{ api_folder_path }}`
- `DOMAIN_NAME` → `{{ domain_name }}`
- `DOMAIN_JOIN` (credential) → `{{ domain_join_user }}` / `{{ domain_join_password }}` (Vault)

---

## Checks for the Migration

**Files to verify**:
- `{{ admin_path }}` — Must exist as a directory after playbook run
- `{{ api_folder_path }}` — Must NOT exist after playbook run

**Registry keys**:
- `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`
  - `ConsentPromptBehaviorAdmin` = `2`
  - `EnableLUA` = `1`
  - `PromptOnSecureDesktop` = `1`
- `HKLM:\SOFTWARE\Microsoft\PowerShell\1\ShellIds\Microsoft.PowerShell`
  - `ExecutionPolicy` = `RemoteSigned`

**Services to check**:
- `vmms` (Hyper-V Virtual Machine Management) — Running
- `hgs` / `IHSvc` (Host Guardian) — Running
- `ClusSvc` (Cluster Service) — Running (if cluster is formed)
- `MpsSvc` (Windows Firewall) — Running

**Firewall rules**: None explicitly defined in this script.

**Hyper-V vSwitch to verify**:
- Switch name: `OS-Traffic`
- Type: External
- SET enabled: `True`
- Management OS: `True`
- Bound adapters: `NIC1`, `NIC2`

**Domain membership**:
- Node must be joined to `{{ domain_name }}`

---

## Pre-flight Checks

```yaml
# 1. Verify all Windows Features are installed
- name: Check Windows Features
  ansible.windows.win_shell: |
    $features = @('Hyper-V','BitLocker','HostGuardian','RSAT-Shielded-VM-Tools',
      'RSAT-Hyper-V-Tools','Hyper-V-Tools','Hyper-V-PowerShell','Failover-Clustering',
      'RSAT-Clustering-PowerShell','FS-FileServer','NetworkVirtualization',
      'RSAT-Clustering-Mgmt','Data-Center-Bridging')
    foreach ($f in $features) {
      $r = Get-WindowsFeature -Name $f
      Write-Output "$($r.Name): $($r.InstallState)"
    }

# 2. Verify vSwitch exists with correct configuration
- name: Check OS-Traffic vSwitch
  ansible.windows.win_shell: |
    $sw = Get-VMSwitch -Name 'OS-Traffic' -ErrorAction SilentlyContinue
    if ($sw) {
      Write-Output "Switch: $($sw.Name), Type: $($sw.SwitchType), SET: $($sw.EmbeddedTeamingEnabled), MgmtOS: $($sw.AllowManagementOS)"
    } else { Write-Output "MISSING: OS-Traffic switch not found" }

# 3. Verify domain membership
- name: Check domain membership
  ansible.windows.win_shell: |
    (Get-WmiObject Win32_ComputerSystem).Domain

# 4. Verify timezone
- name: Check timezone
  ansible.windows.win_shell: |
    (Get-TimeZone).Id

# 5. Verify admin folder exists
- name: Check admin folder
  ansible.windows.win_stat:
    path: "{{ admin_path }}"
  register: admin_folder_stat

# 6. Verify API folder is absent
- name: Check API folder is removed
  ansible.windows.win_stat:
    path: "{{ api_folder_path }}"
  register: api_folder_stat

# 7. Verify UAC registry settings
- name: Check UAC registry
  ansible.windows.win_reg_stat:
    path: HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System
    name: ConsentPromptBehaviorAdmin
  register: uac_reg

# 8. Verify PowerShell execution policy
- name: Check execution policy
  ansible.windows.win_shell: |
    Get-ExecutionPolicy -Scope LocalMachine
```

---

## Recommended Ansible Playbook Task Order

> The following order mirrors the DSC dependency chain and ensures correct sequencing:

1. Set UAC registry keys
2. Set timezone
3. Set PowerShell execution policy
4. Create admin folder
5. Remove API registration folder
6. **Install all 13 Windows Features** (with `include_sub_features: true`)
7. **Reboot** (required for Hyper-V and HostGuardian)
8. **Join Active Directory domain** (requires reboot)
9. **Reboot** (post-domain-join)
10. **Create OS-Traffic SET vSwitch** (requires Hyper-V to be active post-reboot)

> ⚠️ **Dell-specific note**: On Dell hardware, ensure the physical NICs `NIC1` and `NIC2` are correctly named in Windows Device Manager before running the vSwitch task. Dell servers may use names like `Embedded NIC 1 Port 1 Partitioning 1` — verify and adjust `NetAdapterName` values accordingly in your Ansible variables.