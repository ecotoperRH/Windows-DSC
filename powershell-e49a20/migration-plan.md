# MIGRATION FROM POWERSHELL DSC TO ANSIBLE

## Executive Summary

This repository implements an **Enhanced Security Administrative Forest (ESAE / "Red Forest")** using **PowerShell Desired State Configuration (DSC)** hosted on **Azure Automation**. The environment targets Windows Server 2019 and enforces an aggressive, bastion-grade security posture across Active Directory, Host Guardian Service (HGS), Hyper-V hypervisors, SQL Server, and domain-joined member servers.

The migration scope covers **8 DSC Configuration scripts** (the primary automation units), **3 HGS imperative setup scripts**, **7 Group Policy Object (GPO) backups**, **7 operational runbooks**, and **3 one-time configuration artefacts**. All credentials are currently stored in **Azure Automation Vault** and retrieved at runtime via `Get-AutomationPSCredential` / `Get-AutomationVariable`.

**Estimated migration complexity: HIGH**
- Windows-native features (AD DS, HGS, Hyper-V, S2D, Failover Clustering) require the `ansible.windows`, `community.windows`, and `microsoft.ad` Ansible collections.
- Azure Automation's pull-server model must be replaced with Ansible's push model (or AWX/AAP for a comparable pull-like workflow).
- Secrets management must be re-platformed from Azure Automation Vault to **Ansible Vault** or an external secrets backend (HashiCorp Vault, Azure Key Vault lookup plugin).
- Several imperative, interactive scripts (HGS setup, NIC configuration) require careful decomposition into idempotent Ansible tasks.

**Estimated timeline: 10–16 weeks** for a team of 2–3 engineers with Windows/Ansible experience.

---

## Module Migration Plan

This repository contains PowerShell DSC Configurations and supporting scripts that need individual migration planning:

### MODULE INVENTORY

**CRITICAL PATH VERIFICATION:**
Only modules whose paths were confirmed in the provided repository tree or via `read_file` are listed below.

---

- **ActiveDirectoryBuild**
  - Description: Builds the first Domain Controller and establishes a new Active Directory forest from scratch. Configures UAC, timezone, PowerShell execution policy, installs AD DS / DNS / BitLocker / RSAT features, creates the forest (`ADDomain`), enforces a strict domain password and lockout policy (30-char minimum, 50-attempt lockout), creates AD replication sites (SE1, LAS, ORG), provisions a KDS Root Key for gMSA, pre-stages all Tier 0/1/2 admin security groups, PAW Users group, DHCP admin group, domain-join delegation groups, SQL admin groups, gMSA accounts for SQL and SCVMM, Cluster Name Objects (CNO) for six S2D clusters, DNS conditional forwarders and reverse-lookup zones, and bootstraps three named admin user accounts.
  - Path: `ActiveDirectoryBuild.ps1`
  - Technology: PowerShell DSC (Azure Automation)
  - Key Features: `ADDomain` forest creation, `ADDomainDefaultPasswordPolicy`, `ADReplicationSite`, `ADKDSKey`, `ADGroup` / `ADManagedServiceAccount` / `ADOrganizationalUnit` / `ADComputer` / `ADUser` bulk provisioning, `xDnsServer` conditional forwarders and AD-integrated reverse zones, ESAE Tier 0/1/2 OU structure

---

- **ActiveDirectoryHub**
  - Description: Promotes an additional Domain Controller and joins it to an existing forest (hub DC role). Applies the same base OS hardening as `ActiveDirectoryBuild` (UAC, timezone, execution policy, admin folder), installs AD DS / DNS / BitLocker / RSAT features, waits for the forest to be reachable (`WaitForADDomain`), then promotes the node as a replica DC (`ADDomainController`), and monitors the NTDS service.
  - Path: `ActiveDirectoryHub.ps1`
  - Technology: PowerShell DSC (Azure Automation)
  - Key Features: `WaitForADDomain` pre-check, `ADDomainController` forest join, NTDS service monitoring, identical base OS hardening baseline as `ActiveDirectoryBuild`

---

- **AzureConnect**
  - Description: Generic domain-join baseline for Azure-hosted Windows nodes. Applies base OS hardening (UAC `AlwaysNotify`, Pacific timezone, `RemoteSigned` execution policy), creates the admin folder, removes the DSC API registration folder post-onboarding, installs BitLocker, and joins the machine to the Active Directory domain using `xDSCDomainjoin`.
  - Path: `AzureConnect.ps1`
  - Technology: PowerShell DSC (Azure Automation)
  - Key Features: `xDSCDomainjoin` domain join, BitLocker feature install, API folder cleanup (post-registration secret hygiene), shared base OS hardening pattern

---

- **MemberServer**
  - Description: Identical in structure and purpose to `AzureConnect` — a domain-join baseline for generic Windows member servers. Applies the same base OS hardening, installs BitLocker, and joins the domain via `xDSCDomainjoin`. Functionally a duplicate of `AzureConnect`; likely differentiated by node targeting or Azure Automation node assignment rather than code content.
  - Path: `MemberServer.ps1`
  - Technology: PowerShell DSC (Azure Automation)
  - Key Features: `xDSCDomainjoin` domain join, BitLocker, shared base OS hardening pattern

---

- **MemberServerSQL**
  - Description: Provisions a SQL Server member server. Extends the base member-server pattern with SQL Server 2016 installation (`SqlSetup`, named instance `SCVMMSQL`), .NET Framework 4.5 feature install, gMSA-based SQL and Agent service accounts, custom TCP port 50001 (`SqlServerNetwork`), and a domain-profile inbound firewall rule for SQL connectivity (`FireWall`). Requires pre-staged AD group membership and gMSA delegation before execution.
  - Path: `MemberServerSQL.ps1`
  - Technology: PowerShell DSC (Azure Automation)
  - Key Features: `SqlSetup` (SQL Engine, named instance, custom data/log/backup paths), `SqlServerNetwork` TCP port configuration, `FireWall` DSC resource, gMSA service account via `SqlServerDsc`, `NetworkingDsc`, hardcoded SQL installer source path (`C:\Admin\SQL2016_x64_ENU`)

---

- **RedForestBuild**
  - Description: Configures the Host Guardian Service (HGS) "Red Forest" — a dedicated, isolated AD forest for the Guarded Fabric. Installs `HostGuardianServiceRole` and `RSAT-Shielded-VM-Tools`, waits for the HGS forest (`WaitForADDomain`), monitors NTDS, enforces the same strict password/lockout policy as the main forest, creates HGS-specific AD replication sites (SE1, LAS1), provisions HGS Users and Admins security groups, and configures DNS forwarders and a reverse-lookup zone. Must be run **after** the HGS forest is created imperatively via `host-guardian-service/` scripts.
  - Path: `RedForestBuild.ps1`
  - Technology: PowerShell DSC (Azure Automation)
  - Key Features: `GuardedFabricTools` DSC resource, `HostGuardianServiceRole` feature, HGS-specific AD groups, DNS forwarder to `10.78.0.10`, hardcoded domain name `cool-name.net`, dependency on imperative HGS bootstrap scripts

---

- **S2DHypervisorDell**
  - Description: Configures a Dell server as a Storage Spaces Direct (S2D) Hyper-V cluster node. Installs Hyper-V, BitLocker, HostGuardian, Shielded VM tools, Failover Clustering, Scale-Out File Server, Network Virtualization, and Data Center Bridging features. Joins the domain and creates an external virtual switch (`OS-Traffic`) with SET (Switch Embedded Teaming) across two NICs (`NIC1`, `NIC2`). Intended for multi-node S2D cluster deployment.
  - Path: `S2DHypervisorDell.ps1`
  - Technology: PowerShell DSC (Azure Automation)
  - Key Features: `xHyper-V` (`xVMSwitch` with SET teaming on NIC1+NIC2), `GuardedFabricTools`, full S2D feature set (Failover-Clustering, FS-FileServer, NetworkVirtualization, Data-Center-Bridging), `HostGuardian` feature

---

- **StandAloneHypervisorDell**
  - Description: Configures a Dell server as a standalone (non-clustered) Hyper-V host. Identical feature set to `S2DHypervisorDell` but the external virtual switch uses only a single NIC (`NIC1`) rather than a two-NIC SET team. Suitable for single-node or development Hyper-V deployments.
  - Path: `StandAloneHypervisorDell.ps1`
  - Technology: PowerShell DSC (Azure Automation)
  - Key Features: `xVMSwitch` with single NIC (`NIC1`), same full Hyper-V / Guarded Fabric feature set as `S2DHypervisorDell`, `GuardedFabricTools`

---

### Infrastructure Files

- `host-guardian-service/Install-HGS-StepOne.ps1`: Imperative bootstrap — enables PSRemoting/WinRM, installs a root CA PEM certificate into the enterprise trust store, installs `HostGuardianServiceRole` and DNS, renames the computer, and reboots. **Must run before DSC.** Migration consideration: convert to Ansible `win_feature`, `win_certificate`, `win_shell`, and `win_reboot` tasks.
- `host-guardian-service/Install-HGS-StepTwo.ps1`: Imperative bootstrap — imports `HgsServer` module and calls `Install-HGSServer` to create the HGS domain with a DSRM password (interactive prompt). Migration consideration: DSRM password must be sourced from Ansible Vault; `win_shell` task with `no_log: true`.
- `host-guardian-service/Install-HGS-StepThree.ps1`: Imperative bootstrap — calls `Initialize-HgsServer` with PFX certificate paths and passwords to configure TPM attestation, signing, and encryption certificates; runs `Get-HGSTrace` diagnostics. Migration consideration: PFX paths and passwords must come from Ansible Vault; certificate files must be distributed via `win_copy`.
- `group-policy-baseline/ImportGPOBulk.ps1`: Bulk-imports 7 GPO backups from the `group-policy-baseline/` directory into Active Directory using `Import-GPO`. Migration consideration: Ansible `microsoft.ad` collection has no native GPO import module; this will require a `win_shell` task wrapping the existing script, or a custom Ansible module.
- `group-policy-baseline/manifest.xml`: GPO backup manifest identifying 7 GPOs: *DC Virtualization Based Security*, *Credential Guard*, *Defender Antivirus*, *Default Workstation Policy*, *Default Member Servers Policy*, *Default Domain Policy*, *Default Domain Controllers Policy*. All are tagged "Baseline 2019 with Modifications" and sourced from `temp.local`. Migration consideration: GPO content must be reviewed for domain-name references (`temp.local`) before import into the target domain.
- `group-policy-baseline/{GUID}/` (×7): Individual GPO backup folders containing `Backup.xml`, `gpreport.xml`, `registry.pol`, and where applicable `audit.csv` and `GptTmpl.inf` (security templates). Migration consideration: `registry.pol` and `GptTmpl.inf` settings should be audited and replicated as Ansible `win_regedit` and `win_security_policy` tasks where possible, to avoid GPO dependency.
- `code-integrity/Audit/AllowMicrosoft_DenyByPassApps_Audit.xml`: WDAC (Windows Defender Application Control) policy in audit mode — allows Microsoft-signed code, denies bypass applications. Migration consideration: deploy via `win_copy` + `win_shell` (`ConvertFrom-CIPolicy` / `CiTool`).
- `code-integrity/Enforce/AllowMicrosoft_DenyByPassApps_Enforce.xml`: WDAC enforcement-mode policy. Migration consideration: same as audit policy; enforce only after audit validation in target environment.
- `one-time-config/special-use-case/Office365-TrustedSites.ps1` + `Office365-TrustedSites.xml`: Adds Office 365 URLs to the IE/Edge Trusted Sites zone via registry (`HKCU:\...ZoneMap\Domains`). Migration consideration: convert to `win_regedit` tasks driven by a variable list of URLs, or deploy via GPO preference.
- `one-time-config/special-use-case/SetStrongCrypto.reg`: Registry file enforcing `SchUseStrongCrypto=1` for .NET Framework 4.x (both 32-bit and 64-bit) to disable weak TLS. Migration consideration: direct replacement with two `win_regedit` tasks.
- `one-time-config/dscmetaconfigs-dev/placeholder-dev.mof` + `one-time-config/dscmetaconfigs-live/placeholder-live.mof`: Azure Automation DSC onboarding MOF files (LCM metaconfigurations) for dev and live environments. Migration consideration: these are replaced entirely by Ansible inventory and connection configuration; no direct equivalent needed.
- `runbooks/GenerateDSCConfig.ps1`: Generates Azure Automation DSC onboarding metaconfigs via `Get-AzureRmAutomationDscOnboardingMetaconfig`. Migration consideration: obsolete in Ansible; replaced by inventory onboarding.
- `runbooks/CaptureGuardedHost.ps1`: Captures TPM EKPub (`Get-PlatformIdentifier`) and writes it to a network share for HGS TPM attestation registration. Migration consideration: convert to `win_shell` task with output registered as an Ansible fact; file copy via `fetch`.
- `runbooks/ConfigureSMBNICAdapters.ps1`: Interactive script that removes IPv6 addresses and assigns static IPs to four named Dell NIC slots (`SLOT 2 Port 1–2`, `SLOT 3 Port 1–2`) in the `192.168.2–5.x/24` ranges. Migration consideration: convert to `win_shell` or `ansible.windows.win_powershell` tasks with NIC names and IP octets as inventory variables; remove interactive `Read-Host`.
- `runbooks/GenerateShadowPrinciples.ps1`: Creates ESAE Shadow Principal objects in the Red Forest to grant time-limited privileged access from the production forest. Migration consideration: convert to `microsoft.ad.object` or `win_shell` tasks; TTL-based membership (`<TTL=180,...>`) is AD-specific syntax requiring `win_shell`.
- `runbooks/CaptureWindowsFeatures.ps1`: One-liner that lists all installed Windows features — used as a discovery/audit tool. Migration consideration: replace with `ansible.windows.win_feature` facts or `setup` module output.
- `runbooks/ReviewDangerousDirectoryChangesACL.ps1`: Audits AD ACLs for non-standard principals holding DS-Replication-Get-Changes permissions (DCSync attack surface). Migration consideration: convert to an Ansible `win_shell` task producing a report artifact; no direct Ansible module equivalent.
- `runbooks/GetTrueUrl.ps1`: Utility function that resolves HTTP redirects to their final URL. Migration consideration: not infrastructure automation; retain as a utility script or replace with a URI lookup task.

---

### Target Details

- **Operating System**: Windows Server 2019 (explicitly stated in `README.md` as the minimum supported OS). All DSC resources, features, and GPO baselines reference Server 2019 / Windows 10 1809 equivalents.
- **Virtual Machine Technology**: Microsoft Hyper-V with Shielded VMs on a Guarded Fabric (Storage Spaces Direct cluster). Dell hardware is the physical host platform. Virtual TPM (vTPM) is used for guest attestation.
- **Cloud Platform**: Microsoft Azure — Azure Automation is the DSC pull server and secrets/credential store. `Get-AutomationVariable` and `Get-AutomationPSCredential` are Azure Automation-specific APIs. The `AzureRm` PowerShell module is referenced in runbooks.

---

## Migration Approach

### Key Dependencies to Address

- **PSDesiredStateConfiguration** (built-in): Core DSC engine. Replace with native Ansible `ansible.windows` and `community.windows` collection modules; no direct equivalent needed.
- **xPSDesiredStateConfiguration**: Extended DSC resources (file/package/service). Replace with `ansible.windows.win_file`, `ansible.windows.win_package`, `ansible.windows.win_service`.
- **ComputerManagementDSC** (`TimeZone`, `PowerShellExecutionPolicy`): Replace with `community.windows.win_timezone` and `ansible.windows.win_regedit` (execution policy registry key) or `win_shell`.
- **xSystemSecurity** (`xUAC`): UAC configuration. Replace with `ansible.windows.win_regedit` targeting `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`.
- **ActiveDirectoryDsc** (`ADDomain`, `ADDomainController`, `ADGroup`, `ADUser`, `ADOrganizationalUnit`, `ADComputer`, `ADManagedServiceAccount`, `ADKDSKey`, `ADDomainDefaultPasswordPolicy`, `ADReplicationSite`, `WaitForADDomain`): Replace with `microsoft.ad` collection roles and modules (`microsoft.ad.domain`, `microsoft.ad.domain_controller`, `microsoft.ad.group`, `microsoft.ad.user`, `microsoft.ad.ou`, `microsoft.ad.computer`, `microsoft.ad.object`). Password policy and replication sites may require `win_shell` with AD PowerShell cmdlets.
- **xDnsServer** (`xDnsServerConditionalForwarder`, `xDnsServerForwarder`, `xDnsServerADZone`): Replace with `community.windows.win_dns_zone` where available; remaining resources require `win_shell` with `Add-DnsServerConditionalForwarderZone`, `Set-DnsServerForwarder`, `Add-DnsServerPrimaryZone`.
- **xDSCDomainjoin**: Domain join. Replace with `microsoft.ad.membership` module.
- **SqlServerDsc** (`SqlSetup`, `SqlServerNetwork`): SQL Server installation and network configuration. Replace with `community.windows.win_package` for the installer and `win_shell` for instance configuration, or use the `lowlydba.sqlserver` Ansible collection for idiomatic SQL management.
- **NetworkingDsc** (`FireWall`): Windows Firewall rules. Replace with `ansible.windows.win_firewall_rule`.
- **xHyper-V** (`xVMSwitch`): Hyper-V virtual switch management. Replace with `community.windows.win_hyperv` or `win_shell` with `New-VMSwitch` / `Set-VMSwitchTeam`.
- **GuardedFabricTools**: HGS-specific DSC resources. No Ansible collection equivalent exists; replace with `win_shell` tasks wrapping HGS PowerShell cmdlets (`Initialize-HgsServer`, `Get-HgsTrace`, `Set-HgsClientConfiguration`).
- **Azure Automation** (`Get-AutomationVariable`, `Get-AutomationPSCredential`): The entire secrets and variable injection mechanism. Replace with **Ansible Vault** for static secrets, **group_vars / host_vars** for non-sensitive variables, and optionally the **Azure Key Vault lookup plugin** (`azure.azcollection.azure_keyvault_secret`) to preserve Azure as the secrets backend.
- **AzureRm PowerShell module** (runbooks): Replace with `azure.azcollection` Ansible modules or retain as `win_shell` tasks for Azure-specific operations not yet covered by Ansible collections.

---

### Security Considerations

- **Azure Automation Vault credentials**: All eight DSC configurations retrieve credentials exclusively from Azure Automation Vault at runtime (`Get-AutomationPSCredential`). The following credential types are in use and must be re-platformed:
  - `DEFAULT_DC_CRED` — forest safe-mode administrator password (used in `ActiveDirectoryBuild`)
  - `DOMAIN_CONTROLLER_JOIN` — domain admin account for DC promotion (used in `ActiveDirectoryBuild`, `ActiveDirectoryHub`)
  - `DOMAIN_JOIN` — domain join account (used in `AzureConnect`, `MemberServer`, `MemberServerSQL`, `S2DHypervisorDell`, `StandAloneHypervisorDell`, `ActiveDirectoryHub`)
  - `TEMP_PASSWORD` — placeholder credential for gMSA objects (used in `ActiveDirectoryBuild`)
  - All must be migrated to Ansible Vault encrypted variables or an external vault with `no_log: true` on all tasks that consume them.
- **Hardcoded bogus gMSA password**: `MemberServerSQL.ps1` contains a hardcoded plaintext string `"BogusPasswordWorkaround!1"` used to construct a `PSCredential` for the gMSA account (a known DSC limitation). In Ansible, gMSA accounts do not require a password object; this workaround is eliminated entirely.
- **Hardcoded SQL installer path**: `MemberServerSQL.ps1` hardcodes `C:\Admin\SQL2016_x64_ENU` as the installer source. This path must be parameterised as an Ansible variable and the installer pre-staged or fetched from a secure artifact repository.
- **HGS PFX certificate handling**: `Install-HGS-StepThree.ps1` reads PFX certificate paths and passwords interactively. In Ansible, PFX files must be distributed via `win_copy` with `mode: 0600` and passwords sourced from Ansible Vault with `no_log: true`.
- **Root CA certificate distribution**: Both `ActiveDirectoryBuild.ps1` and `RedForestBuild.ps1` include comments warning that the Root CA must be published to AD (`CertUtil.exe -dspublish`) to prevent TLS proxy interception from breaking DSC. This step must be captured as an explicit Ansible task early in the AD build play.
- **API key / DSC registration folder**: The `API_FOLDER_PATH` variable points to the folder containing the Azure Automation DSC onboarding MOF (which contains the registration key). Both `AzureConnect` and `MemberServer` DSC configs explicitly delete this folder post-registration. In Ansible, the equivalent is ensuring no inventory files or connection tokens are left on managed nodes.
- **WDAC policies**: The `code-integrity/` directory contains both audit and enforcement WDAC policies. The enforcement policy must not be deployed before the audit policy has been validated in the target environment, or it may block legitimate software including Ansible's WinRM communication path.
- **GPO security templates**: `GptTmpl.inf` files in several GPO backups contain security policy settings (audit policy, user rights assignments). These must be reviewed before import to ensure they do not conflict with Ansible's WinRM/PSRemoting requirements (e.g., `Deny log on through Remote Desktop Services` rights).
- **Shadow Principals with TTL**: `GenerateShadowPrinciples.ps1` creates time-limited privileged access objects (`<TTL=180,...>`) in the Red Forest. This is a highly sensitive ESAE operation; Ansible tasks performing this must use `no_log: true` and be gated behind a dedicated privileged playbook with strict RBAC in AWX/AAP.
- **DCSync ACL audit**: `ReviewDangerousDirectoryChangesACL.ps1` checks for non-standard DS-Replication-Get-Changes ACEs. This should be converted to a recurring Ansible audit playbook and its output treated as a security artifact.
- **Strong Crypto registry**: `SetStrongCrypto.reg` enforces TLS 1.2+ for .NET 4.x. This is a straightforward `win_regedit` task but must be applied early in the base OS hardening role, before any .NET-dependent services start.
- **WinRM / PSRemoting exposure**: `Install-HGS-StepOne.ps1` explicitly runs `winrm quickconfig -force` and `Enable-PSRemoting -force`. Ansible requires WinRM to be pre-configured on target nodes; this bootstrap step should be documented as a pre-flight requirement and secured with HTTPS (port 5986) and certificate authentication in production.

---

### Technical Challenges

- **Azure Automation pull model vs. Ansible push model**: The entire repository is designed around Azure Automation as a DSC pull server with node registration. Ansible is push-based. Teams must decide whether to use AWX/Ansible Automation Platform to approximate scheduled drift remediation, or accept a purely on-demand push model.
- **Multi-step, ordered HGS bootstrap**: The HGS setup is explicitly split into three sequential imperative scripts with mandatory reboots between steps. Ansible's `win_reboot` module handles reboots, but the three-step sequence must be carefully modelled as a single playbook with `serial: 1` and reboot handlers, preserving the dependency chain.
- **Interactive `Read-Host` prompts**: `Install-HGS-StepTwo.ps1`, `Install-HGS-StepThree.ps1`, and `ConfigureSMBNICAdapters.ps1` use `Read-Host` for passwords and IP address input. These must be fully replaced with Ansible variables and Vault-encrypted secrets — no interactive prompts are possible in Ansible playbooks.
- **GuardedFabricTools DSC resource**: There is no Ansible collection equivalent for Guarded Fabric / HGS management. All `GuardedFabricTools`-backed DSC resources in `RedForestBuild.ps1`, `S2DHypervisorDell.ps1`, and `StandAloneHypervisorDell.ps1` must be replaced with `ansible.windows.win_shell` tasks wrapping native HGS PowerShell cmdlets, with idempotency guards (`Get-` checks before `Set-`/`Initialize-`).
- **GPO import has no native Ansible module**: The `ImportGPOBulk.ps1` script and the 7 GPO backup folders have no direct Ansible equivalent. Options are: (a) wrap the existing script in a `win_shell` task, (b) write a custom Ansible module, or (c) migrate GPO settings to native Ansible tasks (`win_regedit`, `win_security_policy`, `win_audit_policy_system`) and retire the GPO dependency entirely. Option (c) is preferred for long-term maintainability but is the most labour-intensive.
- **S2D cluster formation**: `S2DHypervisorDell.ps1` installs Failover Clustering features but does not include the `New-Cluster` / `Enable-ClusterS2D` steps (these are likely in separate runbooks or manual steps). The full S2D cluster formation workflow must be discovered and modelled in Ansible before the hypervisor role is considered complete.
- **SQL Server 2016 installer pre-staging**: `MemberServerSQL.ps1` expects the SQL 2016 installer to be pre-extracted at `C:\Admin\SQL2016_x64_ENU`. Ansible must either copy the installer from a file share (`win_copy`) or mount an ISO (`win_disk_image`) before the SQL installation task runs. SQL 2016 is also approaching end of extended support (July 2026) — consider upgrading to SQL 2019 or 2022 during migration.
- **ESAE / Tiered administration complexity**: The AD structure (Tier 0/1/2, PAW Users, CNOs, gMSA accounts, Shadow Principals) is a sophisticated ESAE architecture. Migrating `ActiveDirectoryBuild.ps1` alone involves 30+ discrete AD objects. These should be broken into focused Ansible roles (ad_forest, ad_groups, ad_ous, ad_users, ad_dns, ad_gmsas) to keep playbooks manageable and testable.
- **DSC idempotency vs. Ansible idempotency**: DSC resources enforce state continuously via the LCM. Ansible tasks are idempotent only when written carefully. Each DSC resource block must be translated to an Ansible task with appropriate `creates`, `changed_when`, or `check_mode` guards to preserve the drift-detection behaviour.
- **Hardcoded domain/IP references**: Several files contain hardcoded values (`cool-name.net`, `10.78.x.x` IP ranges, `192.168.2–5.x` NIC subnets, `DC=cool-name,DC=net`). All must be extracted to Ansible inventory variables or group_vars before the playbooks are environment-agnostic.

---

### Migration Order

1. **Base OS Hardening Role** (low risk, high reuse) — Extract the common hardening pattern shared by all 8 DSC configs (UAC, timezone, execution policy, admin folder, API folder cleanup, BitLocker) into a single reusable Ansible role (`role: windows_base_hardening`). This role will be a dependency of every subsequent play.

2. **AzureConnect / MemberServer** (low complexity) — These two nearly-identical configurations are the simplest: domain join + BitLocker + base hardening. Migrate first to validate the WinRM connection model, Ansible Vault credential injection, and `microsoft.ad.membership` module. Serves as the integration test bed.

3. **ActiveDirectoryHub** (moderate complexity) — Adds DC promotion to an existing forest on top of the base pattern. Validates `microsoft.ad.domain_controller` and `WaitForADDomain` equivalents (`wait_for` + AD connectivity check). Lower risk than `ActiveDirectoryBuild` because the forest already exists.

4. **ActiveDirectoryBuild** (high complexity) — The most complex single configuration: forest creation plus 30+ AD objects. Decompose into sub-roles: `ad_forest`, `ad_password_policy`, `ad_sites`, `ad_ous`, `ad_groups`, `ad_users`, `ad_gmsas`, `ad_computers`, `ad_dns`. Migrate and test each sub-role independently before assembling the full play.

5. **MemberServerSQL** (moderate complexity) — SQL Server installation with gMSA and firewall configuration. Validate SQL installer pre-staging strategy and `lowlydba.sqlserver` or `win_shell`-based installation. Address SQL 2016 EOL risk.

6. **StandAloneHypervisorDell** (moderate complexity) — Simpler of the two hypervisor configs (single NIC). Validates `xHyper-V` → `win_shell` / `community.windows` migration and Guarded Fabric feature installation.

7. **S2DHypervisorDell** (high complexity) — Extends standalone hypervisor with SET teaming (dual NIC) and full S2D feature set. Requires discovery and modelling of the cluster formation steps missing from the DSC config.

8. **RedForestBuild + HGS Bootstrap** (highest complexity) — The most security-sensitive and operationally complex component. Migrate the three imperative HGS scripts into a sequenced Ansible playbook with reboot handlers, then migrate the DSC configuration. Requires `GuardedFabricTools` replacement via `win_shell` and careful PFX/certificate handling via Ansible Vault.

9. **Group Policy Baseline** (parallel track, moderate complexity) — Audit all 7 GPO backups and decide per-GPO whether to import via `win_shell` wrapper or decompose into native Ansible tasks. Run in parallel with steps 4–8 as a separate workstream.

10. **Runbooks & One-Time Configs** (low complexity, parallel track) — Convert operational runbooks to Ansible playbooks/roles. `SetStrongCrypto.reg` → `win_regedit`, `Office365-TrustedSites` → `win_regedit`, `ConfigureSMBNICAdapters` → `win_shell` with variables, `ReviewDangerousDirectoryChangesACL` → audit playbook, `GenerateShadowPrinciples` → privileged playbook with `no_log`.

---

### Assumptions

1. **Azure Automation is being retired**: The migration assumes the Azure Automation DSC pull server will be decommissioned and replaced by Ansible (with AWX/AAP for scheduling if drift remediation is required). If Azure Automation is retained alongside Ansible, a dual-management conflict strategy must be defined.
2. **WinRM is pre-enabled on all target nodes**: Ansible requires WinRM (preferably HTTPS on port 5986) to be configured before playbooks can run. The `Install-HGS-StepOne.ps1` bootstrap enables PSRemoting, but for all other node types, a pre-flight WinRM enablement mechanism (e.g., cloud-init, VM extension, or manual step) must be defined.
3. **Azure Key Vault or Ansible Vault will replace Azure Automation Vault**: The exact secrets backend for the migrated environment has not been specified. This plan assumes Ansible Vault as the default, with the Azure Key Vault lookup plugin as an optional enhancement.
4. **SQL Server version upgrade is out of scope**: The plan assumes SQL Server 2016 is retained as-is during migration. If an upgrade to SQL 2019/2022 is desired, it should be scoped as a separate workstream.
5. **The `cool-name` / `cool-name.net` domain names are placeholders**: All hardcoded domain names, IP addresses, and organisational names in the source code are assumed to be sanitised placeholders. Actual production values must be supplied via Ansible inventory variables.
6. **S2D cluster formation steps exist outside this repository**: `S2DHypervisorDell.ps1` only installs features; the actual `New-Cluster` and `Enable-ClusterS2D` commands are not present. It is assumed these steps exist in undiscovered runbooks, manual runbooks, or tribal knowledge and must be discovered before the hypervisor migration is complete.
7. **GPO baseline domain references (`temp.local`) are test artefacts**: The `manifest.xml` and GPO backups reference `temp.local` as the source domain. It is assumed these GPOs were exported from a test environment and the settings are valid for the target domain, but all domain-specific references within `GptTmpl.inf` and `registry.pol` files must be reviewed before import.
8. **The WDAC enforcement policy will not block Ansible's WinRM transport**: The `code-integrity/Enforce/` policy must be validated in audit mode to confirm it does not block `wsmprovhost.exe` or other WinRM/PSRemoting components before enforcement is activated.
9. **HGS certificates (PFX files) are stored securely outside this repository**: `Install-HGS-StepThree.ps1` references `C:\Admin\host-guardian-service\HGS-Certificate.pfx`. This file is not present in the repository (correctly). It is assumed the certificate is stored in a secure vault and will be distributed via Ansible Vault + `win_copy` during migration.
10. **The `one-time-config/dscmetaconfigs-*` MOF files are placeholder stubs**: The `placeholder-dev.mof` and `placeholder-live.mof` files appear to be empty stubs. It is assumed the actual onboarding MOFs are generated at runtime by `GenerateDSCConfig.ps1` and are not committed to source control. These files have no equivalent in Ansible and can be removed.
11. **Team has Windows + Ansible expertise**: The migration plan assumes the executing team is proficient in both Windows Server administration (AD DS, HGS, Hyper-V, SQL Server) and Ansible for Windows (`ansible.windows`, `microsoft.ad`, `community.windows` collections). If this expertise is not available, a training phase of 2–4 weeks should be added to the timeline.
12. **`microsoft.ad` collection is available**: The plan relies on the `microsoft.ad` Ansible collection (released 2023) for AD management. Ansible version 2.15+ and `microsoft.ad` 1.x are assumed. Older environments using only `community.windows.win_domain_*` modules will require additional adaptation effort.
