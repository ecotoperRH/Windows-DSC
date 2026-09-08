---
source-path: StandAloneHypervisorDell.ps1
---

Now I have the full content of the script. Let me provide the complete migration plan.

---

# Migration Plan: StandAloneHypervisorDell.ps1

**TLDR**: This DSC configuration script provisions a **standalone Hyper-V hypervisor host** on a Dell server. It configures base OS settings (UAC, timezone, PowerShell execution policy), creates/removes specific directories, joins the host to an Active Directory domain, installs a comprehensive set of Windows features (Hyper-V, BitLocker, Host Guardian, Failover Clustering, etc.), and creates an external virtual switch with SET (Switch Embedded Teaming) enabled for VM traffic.

---

## Service Type and Configuration

**Service Type**: Hypervisor / Virtualization Host (Hyper-V Standalone Node)

**Key Operations**:
- Set UAC to `AlwaysNotify`
- Set timezone to `Pacific Standard Time`
- Set PowerShell execution policy to `RemoteSigned` (LocalMachine scope)
- Create admin directory at path defined by `ADMIN_PATH`
- Delete API registration folder at path defined by `API_FOLDER_PATH`
- Join the host to an Active Directory domain using credentials from Azure Automation
- Install 13 Windows Features (including all sub-features): `Hyper-V`, `BitLocker`, `HostGuardian`, `RSAT-Shielded-VM-Tools`, `RSAT-Hyper-V-Tools`, `Hyper-V-Tools`, `Hyper-V-PowerShell`, `Failover-Clustering`, `RSAT-Clustering-PowerShell`, `FS-FileServer`, `NetworkVirtualization`, `RSAT-Clustering-Mgmt`, `Data-Center-Bridging`
- Create an external Hyper-V virtual switch named `OS-Traffic` on NIC `NIC1` with Switch Embedded Teaming (SET) and management OS access enabled

---

## File Structure

**Scripts:**
```
StandAloneHypervisorDell.ps1
```

**Modules:**
```
(none — all modules are DSC resource modules imported via Import-DscResource)
```

**DSC Configurations:**
```
StandAloneHypervisorDell.ps1
```

**Data Files:**
```
(none — variables are sourced from Azure Automation at runtime)
```

---

## Module Explanation

The script is a single DSC `Configuration` block named `StandAloneHypervisorDell`. It executes in the following logical order:

### 1. **DSC Resource Imports** (`StandAloneHypervisorDell.ps1`)
- Imports `PSDesiredStateConfiguration` — built-in DSC resources (`File`, `WindowsFeatureSet`)
- Imports `xPSDesiredStateConfiguration` — extended DSC resources
- Imports `xHyper-V` — DSC resources for Hyper-V (`xVMSwitch`)
- Imports `ComputerManagementDSC` — resources for `TimeZone`, `PowerShellExecutionPolicy`
- Imports `xSystemSecurity` — resources for UAC (`xUAC`)
- Imports `xDSCDomainjoin` — resource for domain join (`xDSCDomainjoin`)
- Imports `GuardedFabricTools` — resources for Host Guardian Service
- **Ansible equivalent**: No direct equivalent; these are handled by individual Ansible modules per task.

---

### 2. **Variable Initialization** (`StandAloneHypervisorDell.ps1`)
- `$ADMIN_PATH` — Retrieved from Azure Automation variable store; the path where the admin folder will be created.
- `$API_FOLDER_PATH` — Retrieved from Azure Automation variable store; the path of the API folder to be deleted.
- `$DOMAIN_NAME` — Retrieved from Azure Automation variable store; the Active Directory domain to join.
- `$DOMAIN_JOIN` — Retrieved from Azure Automation credential store; used for domain join authentication.
- **Ansible equivalent**: These become Ansible variables (`vars`, `group_vars`, or `host_vars`). Credentials should be stored in **Ansible Vault**.

---

### 3. **Base OS Settings** (`StandAloneHypervisorDell.ps1` — Node block)

#### 3a. UAC Configuration — `xUAC UAC`
- Sets UAC to `AlwaysNotify` (most restrictive UAC level).
- **Ansible equivalent**: `ansible.windows.win_regedit` — modify the registry key `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`.

#### 3b. Timezone — `TimeZone TimeZoneSet`
- Sets timezone to `Pacific Standard Time`.
- **Ansible equivalent**: `community.windows.win_timezone`

#### 3c. PowerShell Execution Policy — `PowerShellExecutionPolicy PowerShellExecutionPolicySet`
- Sets execution policy to `RemoteSigned` at `LocalMachine` scope.
- **Ansible equivalent**: `ansible.windows.win_shell` with `Set-ExecutionPolicy RemoteSigned -Scope LocalMachine -Force` or `ansible.windows.win_regedit` targeting `HKLM:\SOFTWARE\Microsoft\PowerShell\1\ShellIds\Microsoft.PowerShell`.

#### 3d. Create Admin Folder — `File AdminFolder`
- Ensures the directory at `$ADMIN_PATH` is **present**.
- **Ansible equivalent**: `ansible.windows.win_file` with `state: directory`.

#### 3e. Delete API Folder — `File RemoveAPIFolder`
- Ensures the directory at `$API_FOLDER_PATH` is **absent** (force-deleted including contents).
- **Ansible equivalent**: `ansible.windows.win_file` with `state: absent`.

#### 3f. Domain Join — `xDSCDomainjoin JoinDomain`
- Joins the host to the Active Directory domain `$DOMAIN_NAME` using `$DOMAIN_JOIN` credentials.
- **Ansible equivalent**: `microsoft.ad.membership` (or `ansible.windows.win_domain_membership`)

---

### 4. **Install Services** (`StandAloneHypervisorDell.ps1` — Node block)

#### 4a. Windows Features — `WindowsFeatureSet InstallFeatures`
Installs all of the following features with all sub-features included:

| # | Feature Name | Purpose |
|---|---|---|
| 1 | `Hyper-V` | Core hypervisor role |
| 2 | `BitLocker` | Drive encryption |
| 3 | `HostGuardian` | Guarded Fabric / Shielded VMs |
| 4 | `RSAT-Shielded-VM-Tools` | Remote tools for shielded VMs |
| 5 | `RSAT-Hyper-V-Tools` | Remote tools for Hyper-V management |
| 6 | `Hyper-V-Tools` | Hyper-V GUI management tools |
| 7 | `Hyper-V-PowerShell` | Hyper-V PowerShell module |
| 8 | `Failover-Clustering` | Windows Failover Clustering |
| 9 | `RSAT-Clustering-PowerShell` | PowerShell tools for clustering |
| 10 | `FS-FileServer` | File Server role |
| 11 | `NetworkVirtualization` | Network virtualization (HNV) |
| 12 | `RSAT-Clustering-Mgmt` | Failover Cluster Manager GUI |
| 13 | `Data-Center-Bridging` | DCB / QoS for storage networking |

- **Ansible equivalent**: `ansible.windows.win_feature` — one task per feature, or a loop over the list. Use `include_all_sub_features: true` and `restart: false` (handle reboot separately).

#### 4b. External Virtual Switch — `xVMSwitch ExternalVSwitch`
- Creates a Hyper-V virtual switch named `OS-Traffic`.
- Type: `External` (bound to a physical NIC).
- `AllowManagementOS = $true` — the host OS shares the switch.
- `NetAdapterName = 'NIC1'` — bound to the physical adapter named `NIC1`.
- `EnableEmbeddedTeaming = $true` — enables Switch Embedded Teaming (SET).
- `DependsOn = '[WindowsFeatureSet]InstallFeatures'` — runs only after all features are installed.
- **Ansible equivalent**: `ansible.windows.win_shell` using `New-VMSwitch` PowerShell cmdlet (no native Ansible module exists for Hyper-V virtual switches).

---

## PowerShell to Ansible Mapping

| PowerShell / DSC Operation | Ansible Module | Notes |
|---|---|---|
| `Import-DscResource` | N/A | Handled implicitly by individual Ansible modules |
| `Get-AutomationVariable` | `vars` / `group_vars` / `host_vars` | Store as Ansible variables; use Ansible Vault for secrets |
| `Get-AutomationPSCredential` | `ansible.builtin.vault` / `vars` | Store domain join credentials in Ansible Vault |
| `xUAC` — `AlwaysNotify` | `ansible.windows.win_regedit` | Set `ConsentPromptBehaviorAdmin`, `PromptOnSecureDesktop` registry values |
| `TimeZone` — `Pacific Standard Time` | `community.windows.win_timezone` | `timezone: Pacific Standard Time` |
| `PowerShellExecutionPolicy` — `RemoteSigned` | `ansible.windows.win_shell` | `Set-ExecutionPolicy RemoteSigned -Scope LocalMachine -Force` |
| `File` — `Ensure = Present` (directory) | `ansible.windows.win_file` | `state: directory`, `path: "{{ admin_path }}"` |
| `File` — `Ensure = Absent` (directory) | `ansible.windows.win_file` | `state: absent`, `path: "{{ api_folder_path }}"` |
| `xDSCDomainjoin` | `microsoft.ad.membership` | `dns_domain_name`, `domain_admin_user`, `domain_admin_password` |
| `WindowsFeatureSet` (13 features) | `ansible.windows.win_feature` | Loop over feature list; `include_sub_features: true` |
| `xVMSwitch` — External SET switch | `ansible.windows.win_shell` | `New-VMSwitch -Name 'OS-Traffic' -NetAdapterName 'NIC1' -AllowManagementOS $true -EnableEmbeddedTeaming $true` |
| Reboot after feature install | `ansible.windows.win_reboot` | Required after Hyper-V and other role installations |

---

## Dependencies

**PowerShell DSC Module Dependencies**:
- `PSDesiredStateConfiguration` (built-in)
- `xPSDesiredStateConfiguration`
- `xHyper-V`
- `ComputerManagementDSC`
- `xSystemSecurity`
- `xDSCDomainjoin`
- `GuardedFabricTools`

**Windows Features** (all with sub-features):
- `Hyper-V`, `BitLocker`, `HostGuardian`, `RSAT-Shielded-VM-Tools`, `RSAT-Hyper-V-Tools`, `Hyper-V-Tools`, `Hyper-V-PowerShell`, `Failover-Clustering`, `RSAT-Clustering-PowerShell`, `FS-FileServer`, `NetworkVirtualization`, `RSAT-Clustering-Mgmt`, `Data-Center-Bridging`

**External Packages**: None (no Chocolatey or Install-Package calls)

**Credentials / Secrets**:
- `DOMAIN_JOIN` credential (from Azure Automation) — must be migrated to Ansible Vault

**Runtime Variables** (from Azure Automation):
- `ADMIN_PATH` — path for admin folder
- `API_FOLDER_PATH` — path for API folder to remove
- `DOMAIN_NAME` — Active Directory domain FQDN

**Service Dependencies**:
- `xVMSwitch` depends on `WindowsFeatureSet` (Hyper-V must be installed before creating the switch)
- Domain join should occur before or after feature installation (no explicit DSC dependency defined, but a reboot after features is typically required before domain join)

---

## Checks for the Migration

**Files to verify**:
- `{{ admin_path }}` — must exist as a directory after playbook run
- `{{ api_folder_path }}` — must NOT exist after playbook run

**Registry keys**:
- `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\ConsentPromptBehaviorAdmin` → `2` (AlwaysNotify)
- `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\PromptOnSecureDesktop` → `1`
- `HKLM:\SOFTWARE\Microsoft\PowerShell\1\ShellIds\Microsoft.PowerShell\ExecutionPolicy` → `RemoteSigned`

**Services to check**:
- `vmms` (Hyper-V Virtual Machine Management) — must be running
- `HvHost` (Hyper-V Host Compute Service) — must be running

**Firewall rules**: None explicitly defined in this script.

**Virtual Switch to verify**:
- `Get-VMSwitch -Name 'OS-Traffic'` — must return an External switch with SET enabled

**Domain membership**:
- `(Get-WmiObject Win32_ComputerSystem).Domain` — must return `$DOMAIN_NAME`

---

## Pre-flight Checks

```yaml
# 1. Verify target NIC 'NIC1' exists before creating the vSwitch
- name: Check NIC1 adapter exists
  ansible.windows.win_shell: |
    Get-NetAdapter -Name 'NIC1' -ErrorAction Stop
  register: nic_check

# 2. Verify domain connectivity before joining
- name: Test domain reachability
  ansible.windows.win_shell: |
    Test-NetConnection -ComputerName "{{ domain_name }}" -Port 389
  register: domain_ping

# 3. Verify sufficient disk space for features
- name: Check disk space on C:
  ansible.windows.win_shell: |
    (Get-PSDrive C).Free / 1GB
  register: disk_space

# 4. Verify OS is Windows Server (Hyper-V not supported on all SKUs)
- name: Check OS SKU supports Hyper-V
  ansible.windows.win_shell: |
    (Get-WmiObject Win32_OperatingSystem).Caption
  register: os_caption

# 5. Post-install: Verify all features are installed
- name: Verify Windows features installed
  ansible.windows.win_feature:
    name: "{{ item }}"
    state: present
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
  check_mode: yes

# 6. Post-install: Verify virtual switch
- name: Verify OS-Traffic vSwitch exists
  ansible.windows.win_shell: |
    $sw = Get-VMSwitch -Name 'OS-Traffic' -ErrorAction Stop
    if ($sw.SwitchType -ne 'External') { throw "Wrong switch type" }
    if (-not $sw.EmbeddedTeamingEnabled) { throw "SET not enabled" }
  register: vswitch_check
```

---

## Important Migration Notes for the Junior Developer

> **⚠️ Reboot Handling**: Installing `Hyper-V` and several other roles **requires a system reboot**. In Ansible, add `ansible.windows.win_reboot` after the `win_feature` tasks. The virtual switch creation (`win_shell` with `New-VMSwitch`) must run **after** the reboot.

> **⚠️ SET (Switch Embedded Teaming)**: The `EnableEmbeddedTeaming = $true` flag means the switch uses SET instead of traditional NIC teaming. The `New-VMSwitch` command must include `-EnableEmbeddedTeaming $true`. If multiple NICs are to be teamed, pass an array to `-NetAdapterName`.

> **⚠️ Domain Join Order**: The DSC script does not explicitly order domain join relative to feature installation. In Ansible, it is recommended to: (1) install features → (2) reboot → (3) join domain → (4) reboot → (5) create vSwitch.

> **⚠️ Azure Automation Variables**: `Get-AutomationVariable` and `Get-AutomationPSCredential` are Azure Automation-specific. In Ansible, replace these with `vars`, `group_vars`, or `host_vars`, and store sensitive values (domain credentials) in **Ansible Vault**.

> **⚠️ UAC Registry Values**: The `xUAC` resource with `AlwaysNotify` maps to these registry values under `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`: `ConsentPromptBehaviorAdmin = 2`, `ConsentPromptBehaviorUser = 3`, `EnableLUA = 1`, `PromptOnSecureDesktop = 1`.