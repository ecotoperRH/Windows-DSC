# Migration Plan: Infrastructure-as-Code to Ansible

## Executive Summary

This document outlines a comprehensive plan to migrate the existing infrastructure-as-code implementation from PowerShell Desired State Configuration (DSC) to Ansible. The current environment is a Microsoft Windows Server-based infrastructure with an emphasis on security, particularly for administrative environments. The infrastructure includes Host Guardian Service (HGS), Active Directory, SQL Server, and Hyper-V components.

The migration to Ansible will provide improved cross-platform capabilities, a more accessible syntax, and better integration with modern CI/CD pipelines while maintaining the security posture of the existing environment. This plan details the inventory of modules to be migrated, infrastructure components, target environment details, migration approach, and implementation order.

## Module Inventory

### Core Infrastructure Modules

| Module Name | Description | Complexity | Dependencies |
|-------------|-------------|------------|--------------|
| ActiveDirectoryBuild | Configures the first domain controller in a forest, sets password policies, creates AD sites, KDS root keys, security groups, OUs, and DNS settings | High | ActiveDirectoryDsc, xDnsServer |
| ActiveDirectoryHub | Adds additional domain controllers to an existing forest | Medium | ActiveDirectoryDsc |
| RedForestBuild | Configures a separate "Red Forest" for enhanced security administrative environment (ESAE) | High | ActiveDirectoryDsc, xDnsServer, GuardedFabricTools |
| AzureConnect | Basic configuration for connecting servers to Azure | Low | xDSCDomainjoin |
| MemberServer | Standard configuration for domain-joined servers | Low | xDSCDomainjoin |
| MemberServerSQL | SQL Server installation and configuration | High | SqlServerDsc, NetworkingDsc |

### Virtualization Modules

| Module Name | Description | Complexity | Dependencies |
|-------------|-------------|------------|--------------|
| S2DHypervisorDell | Configures Dell servers for Storage Spaces Direct (S2D) Hyper-V | Medium | xHyper-V, GuardedFabricTools |
| StandAloneHypervisorDell | Configures standalone Dell Hyper-V hosts | Medium | xHyper-V, GuardedFabricTools |

### Security Modules

| Module Name | Description | Complexity | Dependencies |
|-------------|-------------|------------|--------------|
| Host-Guardian-Service | Three-step process to install and configure Host Guardian Service | Medium | HgsServer PowerShell module |
| Code-Integrity | Audit and enforcement configurations for code integrity | Medium | None |
| Group-Policy-Baseline | Group Policy Object import and configuration | Medium | ActiveDirectory, GroupPolicy modules |

### Utility Scripts

| Script Name | Description | Complexity | Dependencies |
|-------------|-------------|------------|--------------|
| CaptureGuardedHost | Captures TPM information for guarded fabric | Low | None |
| CaptureWindowsFeatures | Exports installed Windows features | Low | None |
| ImportGPOBulk | Imports Group Policy Objects in bulk | Medium | ActiveDirectory, GroupPolicy modules |

## Infrastructure Files

The repository contains several types of infrastructure files:

1. **PowerShell DSC Configurations**: The main configuration files (.ps1) that define the desired state of servers
2. **Host Guardian Service Scripts**: PowerShell scripts for setting up the Host Guardian Service
3. **Group Policy Baselines**: Templates for Group Policy Objects
4. **Code Integrity Policies**: Audit and enforcement configurations for Windows Defender Application Control
5. **One-time Configuration Scripts**: Scripts for initial setup and registration with Azure Automation

## Target Details

### Operating Systems
- Primary: Windows Server 2019 (mentioned as minimum supported OS)
- Potential secondary targets: Linux systems (new capability with Ansible)

### VM Technology
- Hyper-V with Shielded VMs
- Storage Spaces Direct (S2D)

### Cloud Platform
- Microsoft Azure (Azure Automation for DSC)
- Potential for hybrid cloud scenarios

## Migration Approach

### Dependencies and Prerequisites

1. **Ansible Control Node Requirements**:
   - Install Ansible (version 2.10+)
   - Install pywinrm for Windows remote management
   - Install necessary Ansible collections:
     - `ansible.windows`
     - `community.windows`
     - `microsoft.ad`

2. **Target Node Requirements**:
   - WinRM configuration for Windows hosts
   - PowerShell 5.1 or higher
   - .NET Framework 4.5+

3. **Authentication**:
   - Migrate credentials from Azure Automation to Ansible Vault
   - Configure Kerberos authentication for domain-joined servers

### Security Considerations

1. **Credential Management**:
   - Use Ansible Vault to securely store sensitive information
   - Implement least privilege access for Ansible control node
   - Maintain existing security groups and delegation model

2. **Compliance Requirements**:
   - Maintain Host Guardian Service security model
   - Preserve BitLocker encryption requirements
   - Ensure code integrity policies are enforced

3. **Network Security**:
   - Maintain existing firewall configurations
   - Preserve proxy settings for internet access

### Technical Challenges

1. **Complex Active Directory Operations**:
   - Ansible's AD modules may not have feature parity with PowerShell DSC
   - Custom modules may be required for advanced AD operations

2. **Host Guardian Service**:
   - Limited Ansible modules for HGS configuration
   - May require custom modules or direct PowerShell execution

3. **Shielded VMs and TPM**:
   - Specialized Windows security features may require custom Ansible modules
   - Integration with TPM attestation will need careful implementation

4. **Idempotency**:
   - Ensure all Ansible playbooks maintain the idempotent behavior of DSC

## Migration Order

### Phase 1: Foundation and Testing (Weeks 1-2)
1. Set up Ansible control node and test connectivity
2. Create inventory structure and group variables
3. Develop and test basic server configuration playbooks
4. Migrate MemberServer configuration as proof of concept

### Phase 2: Core Infrastructure (Weeks 3-6)
1. Migrate ActiveDirectoryHub configuration
2. Develop SQL Server installation and configuration playbooks
3. Create Hyper-V host configuration playbooks
4. Test and validate core infrastructure playbooks

### Phase 3: Security Components (Weeks 7-10)
1. Develop Host Guardian Service playbooks
2. Implement code integrity policy deployment
3. Create Group Policy management playbooks
4. Test and validate security component playbooks

### Phase 4: Advanced Features and Integration (Weeks 11-14)
1. Migrate ActiveDirectoryBuild and RedForestBuild configurations
2. Develop Storage Spaces Direct configuration
3. Implement Shielded VM management
4. Create comprehensive testing framework

### Phase 5: Finalization and Documentation (Weeks 15-16)
1. Perform full environment testing
2. Develop CI/CD pipeline integration
3. Create comprehensive documentation
4. Train operations team on new Ansible workflows

## Assumptions

1. The target environment will continue to be primarily Windows-based
2. Existing security requirements will remain in place
3. Migration will be performed in parallel with existing infrastructure
4. Sufficient testing environments are available
5. Operations team will receive training on Ansible
6. Azure will continue to be the primary cloud platform

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Feature parity gaps between DSC and Ansible | High | Medium | Develop custom Ansible modules where needed; use win_shell for direct PowerShell execution |
| Security posture degradation | High | Low | Comprehensive testing of security configurations; maintain existing security controls |
| Knowledge gap in operations team | Medium | High | Provide thorough training and documentation; phase implementation |
| Performance issues with Ansible at scale | Medium | Low | Implement Ansible AWX/Tower for better scalability; optimize playbooks |
| Integration issues with existing tools | Medium | Medium | Test integration points early; develop adapters if needed |

## Conclusion

The migration from PowerShell DSC to Ansible represents a significant opportunity to modernize the infrastructure-as-code approach while maintaining the strong security posture of the existing environment. By following this phased approach and addressing the identified challenges, the migration can be completed with minimal disruption while providing improved capabilities for cross-platform management, CI/CD integration, and operational efficiency.

Regular reviews of this migration plan should be conducted throughout the implementation process to address any emerging issues and adjust the approach as needed.