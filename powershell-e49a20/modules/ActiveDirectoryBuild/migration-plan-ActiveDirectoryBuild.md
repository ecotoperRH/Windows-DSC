---
source-path: ActiveDirectoryBuild.ps1
---

Now I have a thorough understanding of all the files. Let me produce the complete migration plan.

---

# Migration Plan: ActiveDirectoryBuild.ps1

**TLDR**: This DSC configuration suite builds and configures a complete Active Directory forest infrastructure from scratch. It covers three distinct roles: (1) **ActiveDirectoryBuild** — the primary forest root Domain Controller, (2) **ActiveDirectoryHub** — additional/hub Domain Controllers that join the existing forest, and (3) **RedForestBuild** — a Host Guardian Service (HGS) "Red Forest" Domain Controller for a Guarded Fabric architecture. Together they install AD DS/DNS Windows features, promote domain controllers, enforce OS baseline settings (UAC, timezone, PowerShell execution policy), configure domain password/lockout policies, create AD replication sites, OUs, security groups, gMSA accounts, computer pre-stage objects (CNOs), DNS forwarders/zones, and administrative users — all sourced from Azure Automation variables and credential assets.

---

## Service Type and Configuration

**Service Type**: Active Directory Domain Services (AD DS) — Primary Forest Domain Controller, Hub Domain Controller, and Host Guardian Service (Red Forest) Domain Controller

**Key Operations**:
- Install Windows Features: `AD-Domain-Services`, `DNS`, `RSAT-AD-PowerShell`, `RSAT-ADDS`, `RSAT-DNS-Server`, `BitLocker`, `RSAT-Feature-Tools-BitLocker-BdeAducExt`
- Promote first Domain Controller and build the AD forest (`ADDomain`)
- Join additional Domain Controllers to the existing forest (`ADDomainController`)
- Install HGS role (`HostGuardianServiceRole`, `RSAT-Shielded-VM-Tools`) and wait for Red Forest
- Enforce OS baseline: UAC = AlwaysNotify, Timezone = Pacific Standard Time, PS ExecutionPolicy = RemoteSigned
- Create admin directory (`$ADMIN_PATH`), remove API registration folder (`$API_FOLDER_PATH`)
- Monitor NTDS service (Automatic / Running)
- Configure domain default password and lockout policy
- Create AD replication sites: `cool-name-SE1`, `cool-name-LAS`, `cool-name-ORG` (primary forest); `Red-Forest-SE1`, `Red-Forest-LAS1` (Red Forest)
- Create KDS Root Key for Group Managed Service Accounts
- Create AD Security Groups (Universal/Security): `Admin-Tier-Zero`, `Admin-Tier-One`, `Admin-Tier-Two`, `PAW-Users`, `Admin-DHCP-Manage`, `Admin-Domain-Join`, `Admin-SQL-Group`, `Admin-gmsa-SQL-Group`, `Cluster-Tier-One`, `Cluster-Tier-Zero`, `Admin-gmsa-SCVMM-Group`, `Admin-HGS-Users`, `Admin-HGS-Admins`
- Create Group Managed Service Accounts (gMSA): `gmsaSVC-SQL`, `gmsaSVC-SCVMM`
- Create Organizational Units (OU hierarchy under `$BASE_DN`)
- Pre-stage Cluster Name Objects (CNOs): `COREADMIN`, `CORESQL`, `CORESQLDR`, `COREPRIMARY`, `CORESECONDARY`, `CORESOFS`
- Configure DNS conditional forwarders, default forwarders, and reverse-lookup AD zones
- Create AD user accounts: `AD-Domain-Join`, `DC-Domain-Join`, `AdminOne`, `AdminTwo`, `AdminThree`

---

## File Structure

**Scripts / DSC Configurations:**
```
ActiveDirectoryBuild.ps1
ActiveDirectoryHub.ps1
RedForestBuild.ps1
```

**Related Scripts (same repository, not directly analyzed):**
```
AzureConnect.ps1
MemberServer.ps1
MemberServerSQL.ps1
S2DHypervisorDell.ps1
StandAloneHypervisorDell.ps1
```

**Supporting Directories (not analyzed in detail):**
```
code-integrity/
group-policy-baseline/
host-guardian-service/
one-time-config/
runbooks/
```

---

## Module Explanation

The three DSC configurations execute in the following logical order. Each must be compiled into a MOF and applied to its respective target node via Azure Automation DSC or `Start-DscConfiguration`.

---

### 1. `ActiveDirectoryBuild.ps1` — Primary Forest Domain Controller

This is the **first** configuration to run. It builds the AD forest from scratch on the first DC.

#### 1a. DSC Resource Imports
- `PSDesiredStateConfiguration` — built-in DSC resources (`File`, `Service`, `WindowsFeatureSet`)
- `xPSDesiredStateConfiguration` — extended DSC resources
- `ComputerManagementDSC` — `TimeZone`, `PowerShellExecutionPolicy`
- `xSystemSecurity` — `xUAC`
- `ActiveDirectoryDsc` — `ADDomain`, `ADDomainDefaultPasswordPolicy`, `ADReplicationSite`, `ADKDSKey`, `ADGroup`, `ADManagedServiceAccount`, `ADOrganizationalUnit`, `ADComputer`, `ADUser`
- `xDnsServer` — `xDnsServerConditionalForwarder`, `xDnsServerForwarder`, `xDnsServerADZone`

**Ansible equivalent**: All module imports are handled implicitly by using the correct Ansible collection modules. Ensure these collections are installed: `ansible.windows`, `community.windows`, `microsoft.ad`.

#### 1b. Variable Initialization (Azure Automation Variables)
| Variable | Source | Purpose |
|---|---|---|
| `$DOMAIN_NAME` | `Get-AutomationVariable "DOMAIN_NAME"` | FQDN of the AD domain |
| `$ADMIN_PATH` | `Get-AutomationVariable "ADMIN_PATH"` | Path for admin scripts folder |
| `$API_FOLDER_PATH` | `Get-AutomationVariable "API_FOLDER_PATH"` | Path to remove after provisioning |
| `$BASE_DN` | `Get-AutomationVariable "BASE_DN"` | Base Distinguished Name (e.g., `DC=corp,DC=example,DC=com`) |

**Ansible equivalent**: Define these as Ansible variables in `group_vars/` or `host_vars/`, or pass via `--extra-vars`.

#### 1c. Credential Initialization (Azure Automation Credentials)
| Credential | Source | Purpose |
|---|---|---|
| `$DEFAULT_DC_CRED` | `Get-AutomationPSCredential "DEFAULT_DC_CRED"` | DC admin + SafeMode password |
| `$DOMAIN_CONTROLLER_JOIN` | `Get-AutomationPSCredential "DOMAIN_CONTROLLER_JOIN"` | DC join credential |
| `$DOMAIN_JOIN` | `Get-AutomationPSCredential "DOMAIN_JOIN"` | Domain join service account password |
| `$TEMP_PASSWORD` | `Get-AutomationPSCredential "TEMP_PASSWORD"` | Temporary password for new admin users |

**Ansible equivalent**: Store in Ansible Vault. Reference via `vars` or `vault`-encrypted variable files.

#### 1d. Base OS Settings

**Step 1 — UAC: Set to AlwaysNotify**
- DSC: `xUAC { Setting = 'AlwaysNotify' }`
- Ansible: `ansible.windows.win_regedit` — set `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` keys `ConsentPromptBehaviorAdmin = 2`, `PromptOnSecureDesktop = 1`

**Step 2 — Timezone: Pacific Standard Time**
- DSC: `TimeZone { TimeZone = 'Pacific Standard Time' }`
- Ansible: `community.windows.win_timezone` — `timezone: "Pacific Standard Time"`

**Step 3 — PowerShell Execution Policy: RemoteSigned (LocalMachine scope)**
- DSC: `PowerShellExecutionPolicy { ExecutionPolicyScope = 'LocalMachine'; ExecutionPolicy = 'RemoteSigned' }`
- Ansible: `ansible.windows.win_shell` — `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope LocalMachine -Force`

**Step 4 — Create Admin Folder**
- DSC: `File AdminFolder { Ensure = 'Present'; Type = 'Directory'; DestinationPath = $ADMIN_PATH }`
- Ansible: `ansible.windows.win_file` — `path: "{{ admin_path }}"`, `state: directory`

**Step 5 — Remove API Registration Folder**
- DSC: `File RemoveAPIFolder { Ensure = 'Absent'; Type = 'Directory'; Force = $true; DestinationPath = $API_FOLDER_PATH }`
- Ansible: `ansible.windows.win_file` — `path: "{{ api_folder_path }}"`, `state: absent`

#### 1e. Install Windows Features

**Step 6 — Install all AD/DNS/BitLocker features**
- DSC: `WindowsFeatureSet InstallFeatures { Name = $Features; Ensure = 'Present'; IncludeAllSubFeature = $true }`
- Features installed (expand the `$Features` array):
  - `AD-Domain-Services`
  - `DNS`
  - `RSAT-AD-PowerShell`
  - `RSAT-ADDS`
  - `RSAT-DNS-Server`
  - `BitLocker`
  - `RSAT-Feature-Tools-BitLocker-BdeAducExt`
- Ansible: `ansible.windows.win_feature` — one task per feature, or use a loop with `with_items`. Set `include_sub_features: true` and `include_management_tools: true`.

#### 1f. Build the AD Forest

**Step 7 — Promote first Domain Controller / Create Forest**
- DSC: `ADDomain ForestBuild { DomainName = $DOMAIN_NAME; Credential = $DEFAULT_DC_CRED; SafemodeAdministratorPassword = $DEFAULT_DC_CRED; DependsOn = '[WindowsFeatureSet]InstallFeatures' }`
- Ansible: `microsoft.ad.domain` — `dns_domain_name: "{{ domain_name }}"`, `safe_mode_password: "{{ default_dc_password }}"`. This task will reboot the server automatically.

#### 1g. Monitor NTDS Service

**Step 8 — Ensure NTDS is Running and set to Automatic**
- DSC: `Service NTDSService { Name = 'NTDS'; StartupType = 'Automatic'; State = 'Running'; DependsOn = '[ADDomain]ForestBuild' }`
- Ansible: `ansible.windows.win_service` — `name: NTDS`, `start_mode: auto`, `state: started`

#### 1h. Directory Services Management

**Step 9 — Domain Default Password Policy**
- DSC: `ADDomainDefaultPasswordPolicy DomainPasswordPolicy`
- Settings:
  - `PasswordHistoryCount = 24`
  - `MinPasswordAge = 1440` (minutes = 1 day)
  - `MaxPasswordAge = 525600` (minutes = 365 days)
  - `MinPasswordLength = 30`
  - `ComplexityEnabled = $true`
  - `ReversibleEncryptionEnabled = $false`
  - `LockoutDuration = 15` (minutes)
  - `LockoutObservationWindow = 15` (minutes)
  - `LockoutThreshold = 50`
- Ansible: `microsoft.ad.domain_password_policy` or `ansible.windows.win_shell` with `Set-ADDefaultDomainPasswordPolicy` cmdlet.

**Step 10 — AD Replication Sites**
- DSC creates three replication sites:
  1. `ADReplicationSite cool-name-SE1` — Name: `cool-name-SE1`, `RenameDefaultFirstSiteName = $true` (renames `Default-First-Site-Name`)
  2. `ADReplicationSite cool-name-LAS` — Name: `cool-name-LAS`
  3. `ADReplicationSite cool-name-ORG` — Name: `cool-name-ORG`
- Ansible: `microsoft.ad.object` (type `site`) or `ansible.windows.win_shell` with `New-ADReplicationSite`. For renaming the default site, use `Rename-ADObject` targeting `CN=Default-First-Site-Name,CN=Sites,CN=Configuration,...`.

**Step 11 — KDS Root Key**
- DSC: `ADKDSKey KDSRootKey { Ensure = 'Present'; EffectiveTime = '7/1/2019 09:00'; AllowUnsafeEffectiveTime = $true }`
- Ansible: `ansible.windows.win_shell` — `Add-KdsRootKey -EffectiveTime "7/1/2019 09:00" -Force` (note: `AllowUnsafeEffectiveTime` bypasses the 10-hour wait; use with caution in production)

**Step 12 — AD Security Groups** (all Universal / Security scope, all depend on `[ADDomain]ForestBuild`)

| DSC Resource Name | GroupName | Description |
|---|---|---|
| `TierZero-Group` | `Admin-Tier-Zero` | Tier Zero Admin Group |
| `TierOne-Group` | `Admin-Tier-One` | Tier One Admin Group |
| `TierTwo-Group` | `Admin-Tier-Two` | Tier Two Admin Group |
| `PAWUser-Group` | `PAW-Users` | PAW Users Group |
| `Admin-DHCP-Manage` | `Admin-DHCP-Manage` | DHCP Admin Management Group |
| `Admin-DomJoin-Group` | `Admin-Domain-Join` | Domain Join Delegation Group |
| `Admin-SQL-Group` | `Admin-SQL-Group` | SQL Server Administrative Group |
| `Admin-gmsa-SQL-Group` | `Admin-gmsa-SQL-Group` | SQL gMSA Delegation Group |
| `Cluster-Tier-One` | `Cluster-Tier-One` | CNO Delegation Group |
| `Cluster-Tier-Zero` | `Cluster-Tier-Zero` | CNO Delegation Group |
| `Admin-gmsa-SCVMM-Group` | `Admin-gmsa-SCVMM-Group` | SCVMM gMSA Delegation Group |

- Ansible: `microsoft.ad.group` — `name`, `scope: universal`, `category: security`, `description`, `state: present`

**Step 13 — Group Managed Service Accounts (gMSA)**

| DSC Resource | ServiceAccountName | AccountType | DependsOn |
|---|---|---|---|
| `gmsaSVC-SQL` | `gmsaSVC-SQL` | `Group` | `[ADGroup]Admin-gmsa-SQL-Group` |
| `gmsaSVC-SCVMM` | `gmsaSVC-SCVMM` | `Group` | `[ADGroup]Admin-gmsa-SCVMM-Group` |

- Ansible: `ansible.windows.win_shell` — `New-ADServiceAccount -Name "gmsaSVC-SQL" -DNSHostName "gmsaSVC-SQL.$domain_name" -PrincipalsAllowedToRetrieveManagedPassword "Admin-gmsa-SQL-Group"` (repeat for SCVMM). Alternatively use `microsoft.ad.object` if the collection version supports gMSA.

**Step 14 — Organizational Units**

OU hierarchy (in dependency order — parent OUs must be created before children):

| DSC Resource | OU Name | Parent Path |
|---|---|---|
| `OU-Resources` | `Resources` | `$BASE_DN` |
| `OU-PM` | `PM` | `$BASE_DN` |
| `OU-DHCP` | `DHCP` | `OU=PM,$BASE_DN` |
| `OU-Critical-Systems` | `Critical Systems (TIER 0)` | `OU=Resources,$BASE_DN` |
| `OU-Guarded-Fabric` | `Guarded Fabric (TIER 1)` | `OU=Resources,$BASE_DN` |
| `OU-Applications` | `Applications (TIER 1)` | `OU=Resources,$BASE_DN` |
| `OU-Managed-Devices` | `Managed Devices (TIER 2)` | `OU=Resources,$BASE_DN` |

- Ansible: `microsoft.ad.ou` — `name`, `path`, `state: present`

**Step 15 — Cluster Name Objects (CNO) — Pre-staged Computer Accounts (disabled)**

| DSC Resource | ComputerName | Description |
|---|---|---|
| `CNO-COREADMIN` | `COREADMIN` | S2D Admin Cluster |
| `CNO-CORESQL` | `CORESQL` | S2D SQL Cluster |
| `CNO-CORESQLDR` | `CORESQLDR` | S2D SQL DR Cluster |
| `CNO-COREPRIMARY` | `COREPRIMARY` | S2D Primary Cluster |
| `CNO-CORESECONDARY` | `CORESECONDARY` | S2D Secondary Cluster |
| `CNO-CORESOFS` | `CORESOFS` | Scale Out File Server Cluster |

- All created with `EnabledOnCreation = $false` (disabled computer accounts)
- Ansible: `microsoft.ad.computer` — `name`, `description`, `enabled: false`, `state: present`

**Step 16 — DNS Configuration**

| DSC Resource | Type | Name / Target | Details |
|---|---|---|---|
| `DNS-cool-name` | Conditional Forwarder | `Cool Name` | MasterServers: `10.78.20.219`, `10.78.20.220`; ReplicationScope: Forest |
| `DNS-cool-name.net` | Conditional Forwarder | `cool-name.net` | MasterServers: `10.78.0.11`, `10.78.0.12`, `10.78.8.11`; ReplicationScope: Forest |
| `DNS-Default-Forwarder` | Default Forwarder | N/A | IPAddresses: `10.78.62.176`, `10.78.130.26`; UseRootHint: false |
| `DNS-Default-ARPA` | AD-Integrated Zone | `10.in-addr.arpa` | DynamicUpdate: Secure; ReplicationScope: Forest |

- Ansible: `community.windows.win_dns_zone` and `ansible.windows.win_shell` with `Add-DnsServerConditionalForwarderZone`, `Set-DnsServerForwarder`, `Add-DnsServerPrimaryZone`

**Step 17 — AD User Accounts**

| DSC Resource | UserName | UPN | Description | Path | PasswordNeverExpires |
|---|---|---|---|---|---|
| `Admin-Domain-Join` | `AD-Domain-Join` | N/A | Domain Join Account | Default Users container | `$true` |
| `Admin-DC-Join` | `DC-Domain-Join` | N/A | DC Join Account | Default Users container | `$true` |
| `Admin-User-One-ADM` | `AdminOne` | `AdminOne@cool-name.net` | AdminOne | `OU=Managed Devices (TIER 2),OU=Resources,$BASE_DN` | `$true` (PasswordNeverResets) |
| `Admin-User-Two-ADM` | `AdminTwo` | `AdminTwo@cool-name.net` | AdminTwo | `OU=Managed Devices (TIER 2),OU=Resources,$BASE_DN` | `$true` |
| `Admin-User-Three-ADM` | `AdminThree` | `AdminThree@cool-name.net` | AdminThree | `OU=Managed Devices (TIER 2),OU=Resources,$BASE_DN` | `$true` |

- All users: `Ensure = 'Present'`, `RestoreFromRecycleBin = $true`, `CannotChangePassword = $true`
- `AdminOne/Two/Three` also have `Company = "Cool Name Inc"`, `DisplayName` set
- Ansible: `microsoft.ad.user` — `name`, `upn`, `password`, `description`, `path`, `password_never_expires: true`, `cannot_change_password: true`, `state: present`

---

### 2. `ActiveDirectoryHub.ps1` — Additional / Hub Domain Controller

This configuration runs **after** `ActiveDirectoryBuild.ps1` has completed and the forest is available. It joins a new DC to the existing domain.

#### 2a. DSC Resource Imports
Same as `ActiveDirectoryBuild.ps1` **except** `xDnsServer` is not imported (no DNS zone configuration on hub DCs).

#### 2b. Variables
| Variable | Source |
|---|---|
| `$DOMAIN_NAME` | `Get-AutomationVariable "DOMAIN_NAME"` |
| `$ADMIN_PATH` | `Get-AutomationVariable "ADMIN_PATH"` |
| `$API_FOLDER_PATH` | `Get-AutomationVariable "API_FOLDER_PATH"` |

#### 2c. Credentials
| Credential | Purpose |
|---|---|
| `$DOMAIN_CONTROLLER_JOIN` | DC promotion credential + SafeMode password |
| `$DOMAIN_JOIN` | Used to wait/detect forest availability |

#### 2d. Steps (in execution order)

**Step 1–5**: Identical to `ActiveDirectoryBuild.ps1` — UAC, Timezone, PS ExecutionPolicy, Create Admin Folder, Remove API Folder.

**Step 6 — Install Windows Features** (same `$Features` array as primary build):
- `AD-Domain-Services`, `DNS`, `RSAT-AD-PowerShell`, `RSAT-ADDS`, `RSAT-DNS-Server`, `BitLocker`, `RSAT-Feature-Tools-BitLocker-BdeAducExt`
- Ansible: `ansible.windows.win_feature` with loop

**Step 7 — Wait for Forest Availability**
- DSC: `WaitForADDomain 'WaitForestAvailability' { DomainName = $DOMAIN_NAME; Credential = $DOMAIN_JOIN; DependsOn = '[WindowsFeatureSet]InstallFeatures' }`
- Ansible: `microsoft.ad.domain_controller` has a built-in wait, or use `ansible.windows.win_shell` with a retry loop calling `Test-ADDomain` / `Get-ADDomain` before proceeding.

**Step 8 — Join Domain Controller to Existing Forest**
- DSC: `ADDomainController ForestJoin { DomainName = $DOMAIN_NAME; Credential = $DOMAIN_CONTROLLER_JOIN; SafemodeAdministratorPassword = $DOMAIN_CONTROLLER_JOIN; DependsOn = '[WaitForADDomain]WaitForestAvailability' }`
- Ansible: `microsoft.ad.domain_controller` — `dns_domain_name: "{{ domain_name }}"`, `domain_admin_user`, `domain_admin_password`, `safe_mode_password`. Will trigger a reboot.

**Step 9 — Monitor NTDS Service**
- Same as primary build: `ansible.windows.win_service` — `name: NTDS`, `start_mode: auto`, `state: started`

> **Note**: The Hub configuration explicitly states "Do not add Post-Install Services to a HUB Domain Controller Build" — no additional AD objects are created on hub DCs.

---

### 3. `RedForestBuild.ps1` — Host Guardian Service (HGS) / Red Forest Domain Controller

This configuration runs **after** the HGS server has been manually initialized using `Install-HgsServer`. It configures the Red Forest (a separate, isolated AD forest for Guarded Fabric).

> **Critical Pre-requisite Note** (from script comments): Before applying this DSC, the operator must manually run:
> ```powershell
> Install-WindowsFeature -Name HostGuardianServiceRole -IncludeManagementTools -Restart
> $adminPassword = Read-Host -AsSecureString -Prompt "Enter a password..."
> Install-HgsServer -HgsDomainName 'cool-name.net' -SafeModeAdministratorPassword $adminPassword -Restart
> ```
> And copy PFX certificate files to the system before initializing the first HGS node.

#### 3a. DSC Resource Imports
Same as `ActiveDirectoryBuild.ps1` plus `GuardedFabricTools`.

#### 3b. Variables
| Variable | Value/Source |
|---|---|
| `$ADMIN_PATH` | `Get-AutomationVariable "ADMIN_PATH"` |
| `$API_FOLDER_PATH` | `Get-AutomationVariable "API_FOLDER_PATH"` |
| `$DOMAIN_NAME` | Hardcoded: `"cool-name.net"` |

#### 3c. Features Installed
- `HostGuardianServiceRole`
- `RSAT-Shielded-VM-Tools`

#### 3d. Steps (in execution order)

**Step 1–5**: Identical baseline — UAC, Timezone, PS ExecutionPolicy, Create Admin Folder, Remove API Folder.

**Step 6 — Install HGS Features**
- `HostGuardianServiceRole`, `RSAT-Shielded-VM-Tools`
- Ansible: `ansible.windows.win_feature` with `include_sub_features: true`

**Step 7 — Wait for HGS/Red Forest Domain**
- DSC: `WaitForADDomain HGSForestWait { DomainName = $DOMAIN_NAME }` (no credential — relies on local machine context)
- Ansible: `ansible.windows.win_shell` with retry loop polling `Get-ADDomain -Identity "cool-name.net"`

**Step 8 — Monitor NTDS Service**
- `ansible.windows.win_service` — `name: NTDS`, `start_mode: auto`, `state: started`

**Step 9 — Domain Default Password Policy** (same settings as primary forest)
- `PasswordHistoryCount=24`, `MinPasswordAge=1440`, `MaxPasswordAge=525600`, `MinPasswordLength=30`, `ComplexityEnabled=$true`, `ReversibleEncryptionEnabled=$false`, `LockoutDuration=15`, `LockoutObservationWindow=15`, `LockoutThreshold=50`
- Ansible: `microsoft.ad.domain_password_policy` or `ansible.windows.win_shell`

**Step 10 — AD Replication Sites (Red Forest)**

| DSC Resource | Name | RenameDefault |
|---|---|---|
| `cool-name-SE1` | `Red-Forest-SE1` | `$true` |
| `cool-name-LAS1` | `Red-Forest-LAS1` | `$false` |

- Ansible: `microsoft.ad.object` or `ansible.windows.win_shell` with `New-ADReplicationSite` / `Rename-ADObject`

**Step 11 — AD Security Groups (Red Forest)**

| DSC Resource | GroupName | Description |
|---|---|---|
| `HGS-Users-Group` | `Admin-HGS-Users` | Tier Zero HGS Users Group |
| `HGS-Admins-Group` | `Admin-HGS-Admins` | Tier Zero HGS Admins Group |

- Ansible: `microsoft.ad.group`

**Step 12 — DNS Configuration (Red Forest)**

| DSC Resource | Type | Details |
|---|---|---|
| `DNS-Default-Forwarder` | Default Forwarder | IPAddresses: `10.78.0.10`; UseRootHint: false |
| `DNS-Default-ARPA` | AD Zone | `10.in-addr.arpa`; DynamicUpdate: Secure; ReplicationScope: Forest |

- Ansible: `ansible.windows.win_shell` with `Set-DnsServerForwarder` and `Add-DnsServerPrimaryZone`

---

## PowerShell to Ansible Mapping

| PowerShell / DSC Operation | Ansible Module | Notes |
|---|---|---|
| `Import-DscResource` | N/A | Handled by Ansible collection installation |
| `Get-AutomationVariable` | `vars` / `group_vars` / `host_vars` | Replace with Ansible variables; use Ansible Vault for secrets |
| `Get-AutomationPSCredential` | `ansible.builtin.include_vars` + Ansible Vault | Store credentials in vault-encrypted variable files |
| `xUAC { Setting = 'AlwaysNotify' }` | `ansible.windows.win_regedit` | Keys: `ConsentPromptBehaviorAdmin=2`, `PromptOnSecureDesktop=1` under `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` |
| `TimeZone { TimeZone = 'Pacific Standard Time' }` | `community.windows.win_timezone` | `timezone: "Pacific Standard Time"` |
| `PowerShellExecutionPolicy { ExecutionPolicy = 'RemoteSigned' }` | `ansible.windows.win_shell` | `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope LocalMachine -Force` |
| `File { Ensure = 'Present'; Type = 'Directory' }` | `ansible.windows.win_file` | `state: directory` |
| `File { Ensure = 'Absent'; Type = 'Directory'; Force = $true }` | `ansible.windows.win_file` | `state: absent` |
| `WindowsFeatureSet { Name = $Features; IncludeAllSubFeature = $true }` | `ansible.windows.win_feature` | Loop over feature names; `include_sub_features: true` |
| `ADDomain { DomainName; Credential; SafemodeAdministratorPassword }` | `microsoft.ad.domain` | Promotes first DC; triggers reboot |
| `WaitForADDomain { DomainName; Credential }` | `ansible.windows.win_shell` (retry loop) | Poll `Get-ADDomain` until available |
| `ADDomainController { DomainName; Credential; SafemodeAdministratorPassword }` | `microsoft.ad.domain_controller` | Joins additional DC to existing forest |
| `Service { Name = 'NTDS'; StartupType = 'Automatic'; State = 'Running' }` | `ansible.windows.win_service` | `name: NTDS`, `start_mode: auto`, `state: started` |
| `ADDomainDefaultPasswordPolicy` | `microsoft.ad.domain_password_policy` or `ansible.windows.win_shell` | Set all password/lockout policy attributes |
| `ADReplicationSite { Name; RenameDefaultFirstSiteName = $true }` | `ansible.windows.win_shell` | `Rename-ADObject` for default site; `New-ADReplicationSite` for new sites |
| `ADKDSKey { EffectiveTime; AllowUnsafeEffectiveTime }` | `ansible.windows.win_shell` | `Add-KdsRootKey -EffectiveTime "..." -Force` |
| `ADGroup { GroupName; GroupScope = 'Universal'; Category = 'Security' }` | `microsoft.ad.group` | `scope: universal`, `category: security` |
| `ADManagedServiceAccount { ServiceAccountName; AccountType = 'Group' }` | `ansible.windows.win_shell` | `New-ADServiceAccount` with `-PrincipalsAllowedToRetrieveManagedPassword` |
| `ADOrganizationalUnit { Name; Path }` | `microsoft.ad.ou` | `name`, `path`, `state: present` |
| `ADComputer { ComputerName; EnabledOnCreation = $false }` | `microsoft.ad.computer` | `name`, `enabled: false`, `state: present` |
| `xDnsServerConditionalForwarder { Name; MasterServers; ReplicationScope }` | `ansible.windows.win_shell` | `Add-DnsServerConditionalForwarderZone -Name ... -MasterServers ... -ReplicationScope Forest` |
| `xDnsServerForwarder { IPAddresses; UseRootHint = $false }` | `ansible.windows.win_shell` | `Set-DnsServerForwarder -IPAddress ... -UseRootHint $false` |
| `xDnsServerADZone { Name; DynamicUpdate; ReplicationScope }` | `ansible.windows.win_shell` | `Add-DnsServerPrimaryZone -Name "10.in-addr.arpa" -ReplicationScope Forest -DynamicUpdate Secure` |
| `ADUser { UserName; Password; PasswordNeverExpires; CannotChangePassword }` | `microsoft.ad.user` | `name`, `password`, `password_never_expires`, `cannot_change_password`, `path` |
| `Install-WindowsFeature HostGuardianServiceRole` (manual pre-req) | `ansible.windows.win_feature` | `name: HostGuardianServiceRole`, `include_management_tools: true` |
| `Install-HgsServer` (manual pre-req) | `ansible.windows.win_shell` | `Install-HgsServer -HgsDomainName ... -SafeModeAdministratorPassword ...` |

---

## Dependencies

**PowerShell DSC Module Dependencies**:
- `PSDesiredStateConfiguration` (built-in)
- `xPSDesiredStateConfiguration`
- `ComputerManagementDSC`
- `xSystemSecurity`
- `ActiveDirectoryDsc`
- `xDnsServer`
- `GuardedFabricTools` (RedForestBuild only)

**Ansible Collection Dependencies** (install before running playbooks):
```bash
ansible-galaxy collection install ansible.windows
ansible-galaxy collection install community.windows
ansible-galaxy collection install microsoft.ad
```

**Windows Features** (ActiveDirectoryBuild + ActiveDirectoryHub):
- `AD-Domain-Services`
- `DNS`
- `RSAT-AD-PowerShell`
- `RSAT-ADDS`
- `RSAT-DNS-Server`
- `BitLocker`
- `RSAT-Feature-Tools-BitLocker-BdeAducExt`

**Windows Features** (RedForestBuild):
- `HostGuardianServiceRole`
- `RSAT-Shielded-VM-Tools`

**External Credentials / Secrets** (from Azure Automation — migrate to Ansible Vault):
- `DEFAULT_DC_CRED`
- `DOMAIN_CONTROLLER_JOIN`
- `DOMAIN_JOIN`
- `TEMP_PASSWORD`

**Service Dependencies** (DSC DependsOn chain):
1. `WindowsFeatureSet` → `ADDomain` / `WaitForADDomain`
2. `ADDomain` / `WaitForADDomain` → `ADDomainController` → `NTDS Service`
3. `ADDomain` → All AD objects (groups, OUs, users, DNS, CNOs)
4. `ADGroup Admin-gmsa-SQL-Group` → `ADManagedServiceAccount gmsaSVC-SQL`
5. `ADGroup Admin-gmsa-SCVMM-Group` → `ADManagedServiceAccount gmsaSVC-SCVMM`
6. `ADOrganizationalUnit OU-PM` → `ADOrganizationalUnit OU-DHCP`
7. `ADOrganizationalUnit OU-Resources` → `OU-Critical-Systems`, `OU-Guarded-Fabric`, `OU-Applications`, `OU-Managed-Devices`
8. `ADOrganizationalUnit OU-Managed-Devices` → `ADUser AdminOne/Two/Three`

---

## Checks for the Migration

**Files to verify**:
- `{{ admin_path }}` — directory must exist
- `{{ api_folder_path }}` — directory must be absent/removed

**Registry keys** (UAC settings):
- `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\ConsentPromptBehaviorAdmin` = `2`
- `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\PromptOnSecureDesktop` = `1`
- `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\EnableLUA` = `1`

**Services to check**:
- `NTDS` — must be `Running`, `StartupType: Automatic`
- `DNS` — must be `Running` after feature install
- `ADWS` (Active Directory Web Services) — should auto-start with AD DS

**AD Objects to verify after migration**:
- Domain exists: `Get-ADDomain`
- Password policy: `Get-ADDefaultDomainPasswordPolicy`
- Replication sites: `Get-ADReplicationSite -Filter *`
- KDS Root Key: `Get-KdsRootKey`
- Groups: `Get-ADGroup -Filter * | Where-Object { $_.Name -like "Admin-*" -or $_.Name -like "Cluster-*" -or $_.Name -like "PAW-*" }`
- gMSA accounts: `Get-ADServiceAccount -Filter *`
- OUs: `Get-ADOrganizationalUnit -Filter *`
- CNOs: `Get-ADComputer -Filter { Enabled -eq $false }`
- DNS zones: `Get-DnsServerZone`
- DNS forwarders: `Get-DnsServerForwarder`
- Users: `Get-ADUser -Filter { SamAccountName -like "Admin*" -or SamAccountName -like "DC-*" -or SamAccountName -like "AD-*" }`

**DNS Forwarders to check**:
- Conditional forwarder `Cool Name` → `10.78.20.219`, `10.78.20.220`
- Conditional forwarder `cool-name.net` → `10.78.0.11`, `10.78.0.12`, `10.78.8.11`
- Default forwarder → `10.78.62.176`, `10.78.130.26`
- Reverse zone `10.in-addr.arpa` — AD-integrated, Secure dynamic update

---

## Pre-flight Checks

### Before Running Any Playbook
```powershell
# Verify target OS is Windows Server 2016/2019/2022
(Get-WmiObject Win32_OperatingSystem).Caption

# Verify WinRM is accessible from Ansible controller
Test-WSMan -ComputerName <target_ip>

# Verify required Ansible collections are installed
ansible-galaxy collection list | grep -E "ansible.windows|community.windows|microsoft.ad"
```

### Primary Forest DC (ActiveDirectoryBuild)
```powershell
# Verify AD DS feature is installed
Get-WindowsFeature -Name AD-Domain-Services

# Verify domain was promoted successfully
Get-ADDomain

# Verify NTDS service is running
Get-Service -Name NTDS | Select-Object Status, StartType

# Verify password policy
Get-ADDefaultDomainPasswordPolicy

# Verify all replication sites
Get-ADReplicationSite -Filter * | Select-Object Name

# Verify all groups exist
@('Admin-Tier-Zero','Admin-Tier-One','Admin-Tier-Two','PAW-Users','Admin-DHCP-Manage',
  'Admin-Domain-Join','Admin-SQL-Group','Admin-gmsa-SQL-Group','Cluster-Tier-One',
  'Cluster-Tier-Zero','Admin-gmsa-SCVMM-Group') | ForEach-Object {
    Get-ADGroup -Identity $_ -ErrorAction SilentlyContinue | Select-Object Name, GroupScope
}

# Verify gMSA accounts
Get-ADServiceAccount -Filter * | Select-Object Name, ObjectClass

# Verify OU structure
Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName

# Verify CNOs (disabled computer accounts)
Get-ADComputer -Filter { Enabled -eq $false } | Select-Object Name, Description

# Verify DNS zones and forwarders
Get-DnsServerZone | Select-Object ZoneName, ZoneType, ReplicationScope
Get-DnsServerForwarder
Get-DnsServerConditionalForwarderZone | Select-Object ZoneName, MasterServers

# Verify user accounts
Get-ADUser -Identity 'AD-Domain-Join' -Properties PasswordNeverExpires
Get-ADUser -Identity 'DC-Domain-Join' -Properties PasswordNeverExpires
Get-ADUser -Identity 'AdminOne' -Properties PasswordNeverExpires, CannotChangePassword
Get-ADUser -Identity 'AdminTwo' -Properties PasswordNeverExpires, CannotChangePassword
Get-ADUser -Identity 'AdminThree' -Properties PasswordNeverExpires, CannotChangePassword
```

### Hub Domain Controller (ActiveDirectoryHub)
```powershell
# Verify DC joined the domain
Get-ADDomainController -Filter * | Select-Object Name, Domain, Site

# Verify replication is healthy
repadmin /replsummary
repadmin /showrepl

# Verify NTDS service
Get-Service -Name NTDS | Select-Object Status, StartType
```

### Red Forest / HGS (RedForestBuild)
```powershell
# Verify HGS feature is installed
Get-WindowsFeature -Name HostGuardianServiceRole

# Verify HGS domain (cool-name.net) is accessible
Get-ADDomain -Identity "cool-name.net"

# Verify NTDS service
Get-Service -Name NTDS | Select-Object Status, StartType

# Verify Red Forest replication sites
Get-ADReplicationSite -Filter * | Select-Object Name

# Verify HGS groups
Get-ADGroup -Identity 'Admin-HGS-Users'
Get-ADGroup -Identity 'Admin-HGS-Admins'

# Verify DNS forwarder and reverse zone
Get-DnsServerForwarder
Get-DnsServerZone -Name '10.in-addr.arpa'

# Verify HGS service health
Get-HgsServer
Get-HgsTrace -RunDiagnostics
```

### UAC and OS Baseline (All Nodes)
```powershell
# Verify UAC registry settings
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" |
    Select-Object ConsentPromptBehaviorAdmin, PromptOnSecureDesktop, EnableLUA

# Verify timezone
(Get-TimeZone).Id  # Should return "Pacific Standard Time"

# Verify PowerShell execution policy
Get-ExecutionPolicy -Scope LocalMachine  # Should return "RemoteSigned"

# Verify admin folder exists
Test-Path $ADMIN_PATH  # Should return True

# Verify API folder removed
Test-Path $API_FOLDER_PATH  # Should return False
```