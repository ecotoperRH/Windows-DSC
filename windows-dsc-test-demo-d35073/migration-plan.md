# MIGRATION FROM POWERSHELL DSC TO ANSIBLE

## Executive Summary

This repository contains a Windows-focused infrastructure-as-code implementation using PowerShell Desired State Configuration (DSC) designed for an Enhanced Security Administrative Forest environment. The codebase is primarily designed to work with Azure Automation DSC and targets Windows Server 2019 environments with a strong security posture. The migration to Ansible will require careful planning to maintain the security controls and Windows-specific configurations while leveraging Ansible's cross-platform capabilities.

**Scope**: 8 primary PowerShell DSC configuration scripts, 7 runbooks, multiple Group Policy Objects, and security configuration files
**Complexity**: High - Extensive Windows-specific configurations, security hardening, and Active Directory integration
**Timeline Estimate**: 3-4 months for complete migration with parallel testing

## Module Migration Plan

This repository contains PowerShell DSC configurations that need individual migration planning:

### MODULE INVENTORY

- **ActiveDirectoryBuild**:
    - Description: Primary Active Directory Domain Controller configuration with forest creation, OU structure, security groups, and DNS settings
    - Path: ActiveDirectoryBuild.ps1
    - Technology: PowerShell DSC
    - Key Features: AD Forest creation, OU structure, security groups, DNS configuration, KDS root key, gMSA accounts

- **ActiveDirectoryHub**:
    - Description: Secondary Domain Controller configuration for hub sites
    - Path: ActiveDirectoryHub.ps1
    - Technology: PowerShell DSC
    - Key Features: Domain Controller promotion, DNS configuration, site replication

- **AzureConnect**:
    - Description: Azure connectivity configuration for hybrid cloud scenarios
    - Path: AzureConnect.ps1
    - Technology: PowerShell DSC
    - Key Features: Azure AD Connect, conditional access, SSO configuration

- **MemberServer**:
    - Description: Standard member server configuration with domain join and baseline security settings
    - Path: MemberServer.ps1
    - Technology: PowerShell DSC
    - Key Features: Domain join, BitLocker, security settings, PowerShell execution policy

- **MemberServerSQL**:
    - Description: SQL Server member configuration with specialized SQL settings
    - Path: MemberServerSQL.ps1
    - Technology: PowerShell DSC
    - Key Features: SQL Server installation, configuration, security settings

- **RedForestBuild**:
    - Description: Enhanced Security Administrative Forest (Red Forest) configuration
    - Path: RedForestBuild.ps1
    - Technology: PowerShell DSC
    - Key Features: Tiered administrative model, privileged access workstations, security boundaries

- **S2DHypervisorDell**:
    - Description: Storage Spaces Direct hypervisor configuration for Dell hardware
    - Path: S2DHypervisorDell.ps1
    - Technology: PowerShell DSC
    - Key Features: Storage Spaces Direct, Hyper-V, cluster configuration

- **StandAloneHypervisorDell**:
    - Description: Standalone Hyper-V hypervisor configuration for Dell hardware
    - Path: StandAloneHypervisorDell.ps1
    - Technology: PowerShell DSC
    - Key Features: Hyper-V, storage configuration, network settings

- **Host Guardian Service**:
    - Description: Host Guardian Service for Shielded VMs in a guarded fabric
    - Path: host-guardian-service/
    - Technology: PowerShell Scripts
    - Key Features: Three-step HGS deployment, TPM attestation, key protection

- **Group Policy Baseline**:
    - Description: Security baseline Group Policy Objects for domain security
    - Path: group-policy-baseline/
    - Technology: Group Policy Objects
    - Key Features: Security settings, Windows hardening, administrative templates

- **Code Integrity**:
    - Description: Windows Defender Application Control policies for code integrity
    - Path: code-integrity/
    - Technology: XML Configuration
    - Key Features: Audit and enforcement policies, application allowlisting

- **One-Time Configuration**:
    - Description: Special use case configurations and one-time setup scripts
    - Path: one-time-config/
    - Technology: PowerShell Scripts
    - Key Features: Office 365 trusted sites, registry settings, DSC metadata configs

- **Runbooks**:
    - Description: Automation runbooks for various administrative tasks
    - Path: runbooks/
    - Technology: PowerShell Scripts
    - Key Features: Guarded host capture, Windows feature capture, DSC configuration generation

### Infrastructure Files

- `README.md`: Overview of the repository, Azure Automation DSC information, and setup instructions
- `group-policy-baseline/ImportGPOBulk.ps1`: Script to import Group Policy Objects in bulk
- `group-policy-baseline/manifest.xml`: Manifest file for Group Policy Objects
- `code-integrity/Audit/AllowMicrosoft_DenyByPassApps_Audit.xml`: Code integrity policy in audit mode
- `code-integrity/Enforce/AllowMicrosoft_DenyByPassApps_Enforce.xml`: Code integrity policy in enforcement mode
- `one-time-config/special-use-case/Office365-TrustedSites.ps1`: Script to configure Office 365 trusted sites
- `one-time-config/special-use-case/Office365-TrustedSites.xml`: XML configuration for Office 365 trusted sites
- `one-time-config/special-use-case/SetStrongCrypto.reg`: Registry settings for strong cryptography

### Target Details

Based on the source configuration files:

- **Operating System**: Windows Server 2019 (explicitly mentioned in README.md as minimum supported OS)
- **Virtual Machine Technology**: Hyper-V with Shielded VMs and Host Guardian Service
- **Cloud Platform**: Microsoft Azure (Azure Automation, Azure AD Connect)

## Migration Approach

### Key Dependencies to Address

- **PSDesiredStateConfiguration**: Replace with Ansible Windows modules (win_feature, win_service, etc.)
- **xPSDesiredStateConfiguration**: Replace with Ansible Windows modules and custom modules where needed
- **ComputerManagementDSC**: Replace with Ansible win_hostname, win_domain_membership modules
- **xSystemSecurity**: Replace with Ansible win_security_policy module
- **ActiveDirectoryDsc**: Replace with Ansible win_domain, win_domain_controller, win_domain_group, win_domain_user modules
- **xDnsServer**: Replace with Ansible win_dns_client, win_dns_record modules
- **xDSCDomainjoin**: Replace with Ansible win_domain_membership module
- **Azure Automation**: Replace with Ansible Tower/AWX for orchestration and scheduling

### Security Considerations

- **Active Directory Security**: The repository implements a tiered administrative model (Tier 0, 1, 2) that must be preserved in Ansible roles
  - Migration approach: Create separate Ansible roles for each tier with appropriate privilege separation

- **Credential Management**: The DSC configurations use Azure Automation credential objects
  - Migration approach: Use Ansible Vault for credential storage and reference in playbooks

- **Group Policy Objects**: Multiple GPOs for security baseline
  - Migration approach: Convert GPO settings to Ansible win_group_policy module configurations

- **Code Integrity Policies**: Windows Defender Application Control policies
  - Migration approach: Use Ansible win_copy to deploy XML policies and win_shell to apply them

- **BitLocker Encryption**: Disk encryption configuration
  - Migration approach: Use Ansible win_shell with BitLocker cmdlets or develop custom module

- **Vault/secrets management**:
  - 4 PS Credential objects detected in ActiveDirectoryBuild.ps1: DEFAULT_DC_CRED, DOMAIN_CONTROLLER_JOIN, DOMAIN_JOIN, TEMP_PASSWORD
  - 1 PS Credential object detected in MemberServer.ps1: DOMAIN_JOIN
  - Azure Automation variables used for sensitive configuration

### Technical Challenges

- **PowerShell DSC Resource Translation**: Many DSC resources have specific behaviors that need to be replicated in Ansible
  - Mitigation: Create custom Ansible modules or use win_dsc module as a bridge during migration

- **Azure Automation Integration**: The current solution is tightly integrated with Azure Automation
  - Mitigation: Implement Ansible Tower/AWX with Azure credential types and webhooks

- **Group Policy Management**: Converting GPO settings to Ansible-manageable configurations
  - Mitigation: Use PowerShell scripts with Ansible win_shell to apply GPOs or convert settings to individual Ansible tasks

- **Host Guardian Service**: Complex multi-step deployment process
  - Mitigation: Create a multi-stage Ansible playbook with clear checkpoints and validation

- **Shielded VMs**: Specialized security feature requiring TPM attestation
  - Mitigation: Develop custom Ansible modules or use win_shell with appropriate PowerShell commands

- **Storage Spaces Direct**: Complex storage configuration
  - Mitigation: Create specialized Ansible roles with idempotent tasks for S2D configuration

### Migration Order

1. **MemberServer** (low risk, high value) - Basic server configuration with domain join
2. **Group Policy Baseline** (moderate complexity) - Security settings applied via GPO
3. **One-Time Configuration** (moderate complexity) - Special use case configurations
4. **ActiveDirectoryBuild** (high complexity, dependencies) - Core AD infrastructure
5. **ActiveDirectoryHub** (high complexity, dependencies) - Secondary DC configuration
6. **MemberServerSQL** (high complexity, specialized) - SQL Server configuration
7. **Host Guardian Service** (high complexity, specialized) - HGS deployment
8. **S2DHypervisorDell** and **StandAloneHypervisorDell** (high complexity, hardware-specific) - Hypervisor configurations
9. **RedForestBuild** (highest complexity) - Enhanced Security Administrative Forest
10. **AzureConnect** (high complexity, cloud integration) - Azure AD connectivity

### Assumptions

1. The target environment will continue to be Windows Server 2019 or newer
2. Azure will remain the cloud platform of choice
3. The tiered administrative model will be maintained
4. Ansible Tower/AWX will replace Azure Automation for orchestration
5. The migration will be performed in phases with parallel operation during transition
6. The security posture must be maintained or enhanced during migration
7. Group Policy Objects will need to be converted to equivalent Ansible configurations
8. Some PowerShell scripts may need to be retained and executed via Ansible win_shell
9. Custom Ansible modules may need to be developed for specialized Windows features
10. The Dell hardware-specific configurations will remain relevant
11. The organization has expertise in both PowerShell DSC and Ansible
12. Testing environments are available for validation before production deployment