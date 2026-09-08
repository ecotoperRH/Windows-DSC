---
source-path: MemberServerSQL.ps1
---

I now have the full content of the file. Let me produce the detailed migration plan.

---

# Migration Plan: MemberServerSQL.ps1

**TLDR**: This DSC Configuration (`MemberServerSQL`) provisions a Windows member server as a SQL Server 2016 host. It joins the machine to an Active Directory domain, applies baseline OS hardening (UAC, timezone, PowerShell execution policy), creates/removes specific admin folders, installs the .NET Framework 4.5 Windows feature, installs a named SQL Server 2016 instance (`SCVMMSQL`) using a Group Managed Service Account (gMSA), configures SQL Server to listen on TCP port 50001, and opens that port in the Windows Firewall. All credentials and path variables are sourced from Azure Automation variables and a credential vault.

---

## Service Type and Configuration

**Service Type**: Database Server (SQL Server 2016 — Named Instance)

**Key Operations**:
- Set UAC to `AlwaysNotify`
- Set timezone to `Pacific Standard Time`
- Set PowerShell execution policy to `RemoteSigned` (LocalMachine scope)
- Create admin directory (`$ADMIN_PATH`)
- Delete API registration directory (`$API_FOLDER_PATH`)
- Join the machine to an Active Directory domain (`$DOMAIN_NAME`) using credential `DOMAIN_JOIN`
- Install Windows Feature: `Net-Framework-45-Core` (including all sub-features)
- Install SQL Server 2016 named instance `SCVMMSQL` (SQLENGINE feature only) using gMSA `$SQL_GMSA_ACCOUNT`
- Configure SQL Server TCP protocol on instance `SQL-SVC`, port `50001`, disable dynamic ports, restart service
- Create inbound Windows Firewall rule `AllowSQLConnection` on TCP port `50001` (Domain profile)

---

## File Structure

**Scripts:**
```
MemberServerSQL.ps1
```

**Modules:**
*(None — DSC resources are imported inline via `Import-DscResource`)*

**DSC Configurations:**
```
MemberServerSQL.ps1
```

**Data Files:**
*(None — all variables sourced at runtime from Azure Automation)*

---

## Module Explanation

The scripts perform operations in this order:

### 1. **DSC Resource Imports** (`MemberServerSQL.ps1`, top of Configuration block)

The configuration imports the following DSC resource modules. Each must be present on the Ansible-managed node or the pull/push server before the configuration compiles:

| DSC Module | Purpose |
|---|---|
| `PSDesiredStateConfiguration` | Built-in DSC resources (File, WindowsFeatureSet) |
| `xPSDesiredStateConfiguration` | Extended DSC resources |
| `ComputerManagementDSC` | TimeZone, PowerShellExecutionPolicy resources |
| `xSystemSecurity` | xUAC resource |
| `SqlServerDsc` | SqlSetup, SqlServerNetwork resources |
| `NetworkingDsc` | Firewall resource |
| `xDSCDomainjoin` | Domain join resource |

**Ansible equivalent**: No direct equivalent needed — Ansible modules replace all of these.

---

### 2. **Variable Initialization** (`MemberServerSQL.ps1`, inside Configuration block)

| Variable | Source | Value / Description |
|---|---|---|
| `$SqlInstallerSourcePath` | Hardcoded | `C:\Admin\SQL2016_x64_ENU` — local path to unpacked SQL 2016 installer |
| `$ADMIN_PATH` | Azure Automation Variable | Path for the admin folder to create |
| `$API_FOLDER_PATH` | Azure Automation Variable | Path for the API folder to delete |
| `$DOMAIN_NAME` | Azure Automation Variable | FQDN of the Active Directory domain to join |
| `$SQL_ADMIN_GROUP` | Azure Automation Variable | AD group granted SQL sysadmin rights |
| `$SQL_GMSA_ACCOUNT` | Azure Automation Variable | gMSA account name for SQL Engine and Agent services |
| `$DOMAIN_JOIN` | Azure Automation Credential | Credential used to join the domain |
| `$SqlServiceCredential` | Constructed | PSCredential wrapping `$SQL_GMSA_ACCOUNT` with a bogus password (gMSA workaround) |
| `$Features` | Hardcoded array | `['Net-Framework-45-Core']` |

**Ansible equivalent**: All variables become Ansible `vars` or `group_vars`/`host_vars`. Credentials become Ansible Vault secrets.

---

### 3. **Base OS Settings** (`MemberServerSQL.ps1` — Node block, first section)

#### 3a. UAC Configuration — `xUAC UAC`
- Sets UAC to `AlwaysNotify` (highest UAC level).
- **Ansible equivalent**: `ansible.windows.win_regedit` — modify `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` keys `ConsentPromptBehaviorAdmin`, `PromptOnSecureDesktop`.

#### 3b. Timezone — `TimeZone TimeZoneSet`
- Sets timezone to `Pacific Standard Time`.
- `IsSingleInstance = 'Yes'` (singleton resource).
- **Ansible equivalent**: `community.windows.win_timezone` with `timezone: Pacific Standard Time`.

#### 3c. PowerShell Execution Policy — `PowerShellExecutionPolicy PowerShellExecutionPolicySet`
- Scope: `LocalMachine`
- Policy: `RemoteSigned`
- **Ansible equivalent**: `ansible.windows.win_shell` running `Set-ExecutionPolicy RemoteSigned -Scope LocalMachine -Force`, or `ansible.windows.win_regedit` on `HKLM:\SOFTWARE\Microsoft\PowerShell\1\ShellIds\Microsoft.PowerShell` key `ExecutionPolicy`.

#### 3d. Create Admin Folder — `File AdminFolder`
- `Ensure = 'Present'`, `Type = 'Directory'`
- `DestinationPath = $ADMIN_PATH`
- **Ansible equivalent**: `ansible.windows.win_file` with `path: "{{ admin_path }}"` and `state: directory`.

#### 3e. Delete API Folder — `File RemoveAPIFolder`
- `Ensure = 'Absent'`, `Type = 'Directory'`, `Force = $true`
- `DestinationPath = $API_FOLDER_PATH`
- **Ansible equivalent**: `ansible.windows.win_file` with `path: "{{ api_folder_path }}"` and `state: absent`.

#### 3f. Domain Join — `xDSCDomainjoin JoinDomain`
- Joins the machine to `$DOMAIN_NAME` using credential `$DOMAIN_JOIN`.
- **Ansible equivalent**: `microsoft.ad.membership` (or `ansible.windows.win_domain_membership`) with `dns_domain_name: "{{ domain_name }}"` and `domain_admin_user`/`domain_admin_password` from Vault. Note: this task will trigger a reboot.

---

### 4. **Install Services** (`MemberServerSQL.ps1` — Node block, second section)

#### 4a. Windows Features — `WindowsFeatureSet InstallFeatures`
- Installs: `Net-Framework-45-Core` (with all sub-features).
- **Ansible equivalent**: `ansible.windows.win_feature` with `name: Net-Framework-45-Core` and `include_sub_features: true`.

#### 4b. SQL Server Installation — `SqlSetup InstallNamedInstance-SCVMMSQL`
- `Action = 'Install'`
- `InstanceName = 'SCVMMSQL'`
- `Features = 'SQLENGINE'` (SQL Engine only — no SSAS, SSRS, SSIS)
- `SQLSvcAccount` = gMSA `$SQL_GMSA_ACCOUNT` (bogus password workaround)
- `AgtSvcAccount` = gMSA `$SQL_GMSA_ACCOUNT`
- `SQLSysAdminAccounts = @("$SQL_ADMIN_GROUP")`
- `SQLCollation = 'SQL_Latin1_General_CP1_CI_AS'`
- `InstallSharedDir = 'C:\Program Files\Microsoft SQL Server'`
- `InstallSharedWOWDir = 'C:\Program Files (x86)\Microsoft SQL Server'`
- `InstanceDir = 'C:\Program Files\Microsoft SQL Server'`
- `InstallSQLDataDir = 'C:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\Data'`
- `SQLUserDBDir = 'C:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\Data'`
- `SQLUserDBLogDir = 'C:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\Data'`
- `SQLTempDBDir = 'C:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\Data'`
- `SQLTempDBLogDir = 'C:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\Data'`
- `SQLBackupDir = 'C:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\Backup'`
- `SourcePath = 'C:\Admin\SQL2016_x64_ENU'`
- `UpdateEnabled = $true`
- `ForceReboot = $false`
- `DependsOn = '[xDSCDomainjoin]JoinDomain'` ← **must run after domain join**
- **Ansible equivalent**: `ansible.windows.win_package` pointing to the SQL Server setup.exe with a response file (`.ini`), OR `chocolatey.chocolatey.win_chocolatey` if a Chocolatey SQL package is available. The recommended approach for enterprise SQL installs is `ansible.windows.win_package` with a `ConfigurationFile` (silent install `.ini`). This task must have `notify: reboot` if needed.

#### 4c. SQL TCP Network Configuration — `SqlServerNetwork ChangeTcpIpOnDefaultInstance`
- `InstanceName = 'SQL-SVC'`
- `ProtocolName = 'Tcp'`
- `IsEnabled = $true`
- `TCPDynamicPort = $false`
- `TCPPort = 50001`
- `RestartService = $true`
- `DependsOn = '[SqlSetup]InstallNamedInstance-SCVMMSQL'`
- **Ansible equivalent**: `ansible.windows.win_shell` using `sqlcmd` or PowerShell with SMO (`[Microsoft.SqlServer.Management.Smo.Wmi.ManagedComputer]`) to set the TCP port, followed by `ansible.windows.win_service` to restart `MSSQL$SCVMMSQL` and `SQLAgent$SCVMMSQL`.

#### 4d. Firewall Rule — `FireWall SQLFirewallRule`
- `Name = 'AllowSQLConnection'`
- `DisplayName = 'Allow SQL Connection'`
- `Group = 'DSC Configuration Rules'`
- `Ensure = 'Present'`
- `Enabled = 'True'`
- `Profile = ('Domain')`
- `Direction = 'InBound'`
- `LocalPort = ('50001')`
- `Protocol = 'TCP'`
- `Description = 'Firewall Rule to allow SQL communication'`
- **Ansible equivalent**: `community.windows.win_firewall_rule` with all matching properties.

---

## PowerShell to Ansible Mapping

| PowerShell / DSC Operation | Ansible Module | Notes |
|---|---|---|
| `xUAC` — `AlwaysNotify` | `ansible.windows.win_regedit` | Set `ConsentPromptBehaviorAdmin=2`, `PromptOnSecureDesktop=1` under `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` |
| `TimeZone` — `Pacific Standard Time` | `community.windows.win_timezone` | `timezone: Pacific Standard Time` |
| `PowerShellExecutionPolicy` — `RemoteSigned` / `LocalMachine` | `ansible.windows.win_shell` | `Set-ExecutionPolicy RemoteSigned -Scope LocalMachine -Force` |
| `File` — `Present` / `Directory` (AdminFolder) | `ansible.windows.win_file` | `state: directory` |
| `File` — `Absent` / `Directory` (RemoveAPIFolder) | `ansible.windows.win_file` | `state: absent` |
| `xDSCDomainjoin` — join domain | `microsoft.ad.membership` | Requires reboot handler; use Vault for credentials |
| `WindowsFeatureSet` — `Net-Framework-45-Core` | `ansible.windows.win_feature` | `include_sub_features: true` |
| `SqlSetup` — Install named instance | `ansible.windows.win_package` | Use SQL silent install `.ini` config file; `win_copy` to stage installer |
| `SqlServerNetwork` — TCP port 50001 | `ansible.windows.win_shell` | Use PowerShell/SMO to set TCP port; restart SQL service after |
| `Service restart` (SQL Engine + Agent) | `ansible.windows.win_service` | `name: MSSQL$SCVMMSQL`, `state: restarted` |
| `FireWall SQLFirewallRule` | `community.windows.win_firewall_rule` | All properties map 1:1 |
| `Get-AutomationVariable` | Ansible `vars` / `group_vars` | Store in `group_vars/all.yml` or `host_vars` |
| `Get-AutomationPSCredential` | `ansible.builtin.include_vars` + Ansible Vault | Encrypt with `ansible-vault` |

---

## Dependencies

**PowerShell DSC Module Dependencies**:
- `PSDesiredStateConfiguration` (built-in)
- `xPSDesiredStateConfiguration`
- `ComputerManagementDSC`
- `xSystemSecurity`
- `SqlServerDsc`
- `NetworkingDsc`
- `xDSCDomainjoin`

**Windows Features**:
- `Net-Framework-45-Core` (with all sub-features)

**External Packages / Installers**:
- SQL Server 2016 x64 ENU installer, pre-staged at `C:\Admin\SQL2016_x64_ENU` on the target node

**Active Directory Pre-requisites** (documented in script comments — must be done manually before Ansible run):
1. Add the computer account to `Admin-gmsa-SQL-Group` in AD (grants gMSA usage rights)
2. Add `Admin-SQL-Group` to the Local Administrators group on the target machine
3. Unpack SQL 2016 installation media to `C:\Admin\SQL2016_x64_ENU`

**Service Dependencies** (execution order enforced by `DependsOn`):
1. Domain Join → must complete before SQL install
2. SQL Install → must complete before TCP network configuration
3. TCP network configuration → triggers SQL service restart

**Credentials Required**:
- `DOMAIN_JOIN` — AD account with rights to join machines to the domain
- `SQL_GMSA_ACCOUNT` — gMSA for SQL Engine and SQL Agent services (no password needed at runtime; bogus password is a DSC workaround)

---

## Checks for the Migration

**Files to verify**:
- `{{ admin_path }}` — directory must exist after playbook run
- `{{ api_folder_path }}` — directory must NOT exist after playbook run
- `C:\Admin\SQL2016_x64_ENU\setup.exe` — SQL installer must be pre-staged
- `C:\Program Files\Microsoft SQL Server\` — SQL shared install directory
- `C:\Program Files (x86)\Microsoft SQL Server\` — SQL shared WOW64 directory
- `C:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\Data\` — data/log/tempdb directory
- `C:\Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\Backup\` — backup directory

**Registry Keys**:
- `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`
  - `ConsentPromptBehaviorAdmin` = `2`
  - `PromptOnSecureDesktop` = `1`
- `HKLM:\SOFTWARE\Microsoft\PowerShell\1\ShellIds\Microsoft.PowerShell`
  - `ExecutionPolicy` = `RemoteSigned`

**Services to check**:
- `MSSQL$SCVMMSQL` — SQL Server Engine (named instance), must be Running
- `SQLAgent$SCVMMSQL` — SQL Server Agent (named instance), must be Running
- Both services must be configured to run as `$SQL_GMSA_ACCOUNT`

**Firewall Rules**:
- Rule name: `AllowSQLConnection`
  - Direction: Inbound
  - Protocol: TCP
  - Local Port: 50001
  - Profile: Domain
  - Enabled: True

**Domain Membership**:
- Machine must be joined to `$DOMAIN_NAME`

---

## Pre-flight Checks

Before running the Ansible playbook, validate the following on the target node and in Active Directory:

```powershell
# 1. Verify SQL installer is staged
Test-Path "C:\Admin\SQL2016_x64_ENU\setup.exe"

# 2. Verify gMSA is accessible from this machine
Test-ADServiceAccount -Identity <SQL_GMSA_ACCOUNT_NAME>

# 3. Verify domain connectivity
Test-ComputerSecureChannel -Verbose

# 4. Verify SQL named instance is installed (post-run check)
Get-Service -Name "MSSQL`$SCVMMSQL" | Select-Object Name, Status, StartType

# 5. Verify SQL Agent service (post-run check)
Get-Service -Name "SQLAgent`$SCVMMSQL" | Select-Object Name, Status, StartType

# 6. Verify TCP port 50001 is listening (post-run check)
netstat -ano | findstr ":50001"

# 7. Verify firewall rule exists (post-run check)
Get-NetFirewallRule -Name "AllowSQLConnection" | Get-NetFirewallPortFilter

# 8. Verify .NET Framework 4.5 feature is installed (post-run check)
Get-WindowsFeature -Name "Net-Framework-45-Core"

# 9. Verify timezone (post-run check)
Get-TimeZone

# 10. Verify PowerShell execution policy (post-run check)
Get-ExecutionPolicy -Scope LocalMachine

# 11. Verify admin folder exists (post-run check)
Test-Path $ADMIN_PATH

# 12. Verify API folder is removed (post-run check)
-not (Test-Path $API_FOLDER_PATH)
```

---

## Ansible Playbook Structure Recommendation

```
site.yml                          ← Main playbook entry point
inventory/
  hosts.yml                       ← Target SQL server(s)
group_vars/
  all.yml                         ← Non-secret variables (paths, domain name, etc.)
  all.vault.yml                   ← Ansible Vault: domain_join_user, domain_join_password
roles/
  sql_member_server/
    tasks/
      main.yml                    ← Orchestrates all task files in order
      01_os_baseline.yml          ← UAC, timezone, execution policy
      02_folders.yml              ← Create/remove directories
      03_domain_join.yml          ← AD domain join + reboot handler
      04_windows_features.yml     ← Net-Framework-45-Core
      05_sql_install.yml          ← SQL Server 2016 silent install
      06_sql_network.yml          ← TCP port 50001 configuration
      07_firewall.yml             ← Firewall rule
    handlers/
      main.yml                    ← Reboot handler, SQL service restart handler
    vars/
      main.yml                    ← Role-level defaults
    files/
      sql_install.ini             ← SQL Server silent install configuration file
```

### Key Ansible Variable Definitions (`group_vars/all.yml`)

```yaml
sql_installer_source_path: "C:\\Admin\\SQL2016_x64_ENU"
admin_path: "{{ lookup('env', 'ADMIN_PATH') }}"       # or hardcoded value
api_folder_path: "{{ lookup('env', 'API_FOLDER_PATH') }}"
domain_name: "contoso.local"                           # from Azure Automation var
sql_admin_group: "CONTOSO\\Admin-SQL-Group"            # from Azure Automation var
sql_gmsa_account: "CONTOSO\\sql-gmsa$"                 # from Azure Automation var
sql_instance_name: "SCVMMSQL"
sql_tcp_port: 50001
sql_collation: "SQL_Latin1_General_CP1_CI_AS"
sql_features: "SQLENGINE"
```

### Key Ansible Vault Variables (`group_vars/all.vault.yml`)

```yaml
domain_join_user: "CONTOSO\\svc-domainjoin"
domain_join_password: "SuperSecretPassword"
```

> **⚠️ Important Notes for the Junior Developer:**
> 1. The gMSA (`sql_gmsa_account`) does **not** require a password in Ansible — use `win_shell` with PowerShell SMO or a SQL response file that specifies the gMSA account name directly. The bogus password in the DSC script is a DSC-specific workaround that does **not** apply to Ansible.
> 2. The domain join task **will reboot the machine**. Use `ansible.builtin.reboot` as a handler or inline task after `microsoft.ad.membership`.
> 3. SQL Server silent install via `win_package` requires a pre-built `ConfigurationFile.ini`. Generate this by running the SQL installer interactively once and saving the configuration file from `%ProgramFiles%\Microsoft SQL Server\130\Setup Bootstrap\Log\<timestamp>\ConfigurationFile.ini`.
> 4. The `SqlServerNetwork` DSC resource targets instance name `SQL-SVC` (not `SCVMMSQL`) — this appears to be a configuration inconsistency in the original script. Verify the correct instance name before migrating the TCP port configuration step.