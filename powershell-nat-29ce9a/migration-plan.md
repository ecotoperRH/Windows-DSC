# Migration Plan: PowerShell DSC to Ansible

## Executive Summary

This document outlines a comprehensive plan to migrate the existing Enhanced Security Administrative Forest (ESAF) infrastructure from PowerShell Desired State Configuration (DSC) to Ansible. The current infrastructure is designed for Windows Azure Automation platform with a focus on high security for administrative environments. The migration will preserve all security features while leveraging Ansible's cross-platform capabilities, improved maintainability, and broader community support.

## Current Environment Assessment

### Architecture Overview

The current infrastructure implements an Enhanced Security Administrative Forest using PowerShell DSC with the following key components:

- **Azure Automation DSC**: Cloud-based configuration management for Windows servers
- **Windows Server 2019+**: Minimum required OS version
- **Security-Focused Design**: Aggressive security posture for administrative environments
- **Network Security**: Reverse Proxy agents limiting required ingress firewall to DNS Forwarding
- **Advanced Windows Features**: Host Guardian Service, Hypervisor-Protected Code Integrity (HVCI)

### Key Components Identified

1. **Active Directory Configuration**
   - Domain setup and configuration
   - OU structure creation
   - Group Policy management
   - Security principals and delegation

2. **Server Configuration**
   - Base OS settings (UAC, timezone, execution policy)
   - Windows features installation
   - Domain joining
   - Service monitoring

3. **Security Components**
   - Host Guardian Service (HGS)
   - Code Integrity policies
   - BitLocker configuration
   - Certificate management

4. **Special Use Cases**
   - Office 365 trusted sites configuration
   - Strong crypto settings
   - Guarded host configuration

5. **Integration Points**
   - Azure Automation
   - Azure AD integration
   - Conditional Access

## Migration Strategy

### 1. Module Mapping

| PowerShell DSC Module | Ansible Equivalent | Notes |
|------------------------|---------------------|-------|
| PSDesiredStateConfiguration | ansible.builtin.* | Core Ansible modules will replace basic functionality |
| xPSDesiredStateConfiguration | community.windows.* | Extended Windows functionality |
| ComputerManagementDSC | ansible.windows.win_* | Computer management tasks |
| xSystemSecurity | community.windows.win_security_policy | Security settings |
| ActiveDirectoryDsc | community.windows.win_domain*, community.windows.win_domain_controller | AD management |
| xDnsServer | community.windows.win_dns_* | DNS server configuration |
| xDSCDomainjoin | ansible.windows.win_domain_membership | Domain joining |
| Host Guardian Service | Custom modules/roles | Will require custom implementation |

### 2. Directory Structure

```
ansible-esaf/
├── inventory/
│   ├── hosts.yml                    # Inventory definitions
│   ├── group_vars/                  # Group variables
│   │   ├── all.yml                  # Variables for all hosts
│   │   ├── domain_controllers.yml   # DC-specific variables
│   │   └── member_servers.yml       # Member server variables
│   └── host_vars/                   # Host-specific variables
├── roles/
│   ├── common/                      # Common configurations for all servers
│   ├── active_directory/            # AD domain controller configuration
│   ├── member_server/               # Member server configuration
│   ├── dns_server/                  # DNS server configuration
│   ├── host_guardian_service/       # HGS configuration
│   ├── code_integrity/              # Code integrity policies
│   └── special_use_cases/           # Special configurations
├── playbooks/
│   ├── site.yml                     # Main playbook
│   ├── active_directory_build.yml   # AD setup playbook
│   ├── member_server.yml            # Member server playbook
│   └── hgs_setup.yml                # HGS setup playbook
├── library/                         # Custom modules
├── filter_plugins/                  # Custom filters
├── vars/
│   └── vault.yml                    # Encrypted sensitive variables
└── ansible.cfg                      # Ansible configuration
```

### 3. Variable Management

#### Azure Automation Variables to Ansible Variables

The current implementation uses Azure Automation variables like:
```powershell
$DOMAIN_NAME = Get-AutomationVariable -Name "DOMAIN_NAME"
$ADMIN_PATH = Get-AutomationVariable -Name "ADMIN_PATH"
```

These will be migrated to Ansible variables in group_vars or host_vars:
```yaml
# group_vars/all.yml
domain_name: "example.com"
admin_path: "C:\\Admin"
```

#### Credential Management

Current credentials are retrieved from Azure Vault:
```powershell
$DEFAULT_DC_CRED = Get-AutomationPSCredential -Name 'DEFAULT_DC_CRED'
```

These will be migrated to Ansible Vault:
```yaml
# vars/vault.yml (encrypted)
default_dc_cred:
  username: Administrator
  password: SecurePassword
```

### 4. Feature-Specific Migration Approach

#### Active Directory Configuration

The `ActiveDirectoryBuild.ps1` script will be converted to Ansible roles and playbooks:

```yaml
# playbooks/active_directory_build.yml
- name: Configure Active Directory Domain Controller
  hosts: domain_controllers
  roles:
    - common
    - active_directory
```

```yaml
# roles/active_directory/tasks/main.yml
- name: Install AD DS features
  ansible.windows.win_feature:
    name:
      - AD-Domain-Services
      - DNS
      - RSAT-AD-PowerShell
      - RSAT-ADDS
      - RSAT-DNS-Server
    state: present
    include_sub_features: yes
    include_management_tools: yes

- name: Configure new forest and domain
  community.windows.win_domain:
    dns_domain_name: "{{ domain_name }}"
    safe_mode_password: "{{ default_dc_cred.password }}"
  register: ad_setup

- name: Reboot after AD installation
  ansible.windows.win_reboot:
  when: ad_setup.changed
```

#### Member Server Configuration

The `MemberServer.ps1` script will be converted to:

```yaml
# playbooks/member_server.yml
- name: Configure Member Servers
  hosts: member_servers
  roles:
    - common
    - member_server
```

```yaml
# roles/member_server/tasks/main.yml
- name: Set UAC configuration
  community.windows.win_security_policy:
    name: EnableLUA
    policy_value: 1

- name: Set timezone
  community.windows.win_timezone:
    timezone: "Pacific Standard Time"

- name: Join domain
  ansible.windows.win_domain_membership:
    dns_domain_name: "{{ domain_name }}"
    hostname: "{{ inventory_hostname_short }}"
    username: "{{ domain_join_user }}"
    password: "{{ domain_join_password }}"
    state: domain
  register: domain_join

- name: Reboot after domain join
  ansible.windows.win_reboot:
  when: domain_join.changed
```

#### Host Guardian Service

The HGS scripts will require custom roles:

```yaml
# roles/host_guardian_service/tasks/main.yml
- name: Install HGS role
  ansible.windows.win_feature:
    name: HostGuardianServiceRole
    state: present
    include_management_tools: yes

- name: Install DNS role
  ansible.windows.win_feature:
    name: DNS
    state: present
    include_management_tools: yes

- name: Install root certificate
  community.windows.win_certificate_store:
    path: "{{ root_cert_path }}"
    store_location: LocalMachine
    store_name: Root
    state: present
```

#### Code Integrity Policies

The code integrity XML files will be deployed using:

```yaml
# roles/code_integrity/tasks/main.yml
- name: Copy code integrity policy
  ansible.windows.win_copy:
    src: files/AllowMicrosoft_DenyByPassApps_Audit.xml
    dest: C:\Windows\CodeIntegrity\SIPolicy.p7b
```

### 5. Azure Automation Integration

To replace Azure Automation DSC, we'll implement:

1. **Ansible AWX/Tower**: For web-based management, scheduling, and inventory
2. **CI/CD Pipeline**: Using GitHub Actions or Azure DevOps for automated deployments
3. **State Reporting**: Using AWX/Tower callback or custom reporting scripts

## Technical Challenges and Solutions

### Challenge 1: Windows-Specific Features

**Challenge**: Many features in the current implementation are Windows-specific with no direct Ansible equivalents.

**Solution**: 
- Use community.windows collection for most Windows functionality
- Develop custom modules for specialized Windows features
- Utilize win_shell or win_command for complex PowerShell commands with no Ansible equivalent

### Challenge 2: Azure Automation Integration

**Challenge**: Replacing Azure Automation DSC pull server functionality.

**Solution**:
- Implement Ansible AWX/Tower for centralized management
- Use dynamic inventory scripts to integrate with Azure
- Set up scheduled runs to replace the pull server model with a push model

### Challenge 3: Security Hardening

**Challenge**: Maintaining the high security posture during and after migration.

**Solution**:
- Implement security-focused Ansible roles
- Use ansible-vault for credential management
- Implement proper privilege escalation
- Maintain all existing security policies in Ansible format

### Challenge 4: Host Guardian Service

**Challenge**: Complex HGS configuration has no direct Ansible equivalent.

**Solution**:
- Create a specialized Ansible role for HGS
- Use win_shell to execute complex PowerShell commands where needed
- Implement idempotent checks to ensure configuration is applied correctly

## Migration Phases

### Phase 1: Preparation and Planning (Weeks 1-2)

1. Complete inventory of all servers and configurations
2. Set up Ansible control node and test connectivity
3. Create base directory structure and initial roles
4. Develop and test variable management strategy

### Phase 2: Core Infrastructure Development (Weeks 3-6)

1. Develop common role for base OS settings
2. Create Active Directory role
3. Develop member server role
4. Implement DNS server configuration
5. Test core infrastructure deployment in isolated environment

### Phase 3: Security Components (Weeks 7-10)

1. Implement Host Guardian Service role
2. Create code integrity policy deployment
3. Develop security hardening roles
4. Test security components in isolated environment

### Phase 4: Special Use Cases (Weeks 11-12)

1. Implement Office 365 trusted sites configuration
2. Create strong crypto settings role
3. Develop other special use case roles
4. Test special use cases in isolated environment

### Phase 5: Integration and Testing (Weeks 13-16)

1. Set up Ansible AWX/Tower
2. Implement CI/CD pipeline
3. Integrate with existing monitoring systems
4. Perform comprehensive testing in staging environment

### Phase 6: Deployment and Handover (Weeks 17-20)

1. Deploy to production in phases
2. Monitor for issues and performance
3. Document final implementation
4. Train operations team on new Ansible-based system

## Security Considerations

### Authentication and Authorization

- Use Ansible Vault for credential storage
- Implement least privilege principle for Ansible execution
- Use dedicated service accounts with appropriate permissions

### Network Security

- Maintain existing reverse proxy architecture
- Ensure Ansible control node is properly secured
- Use SSH for Linux and WinRM over HTTPS for Windows communication

### Compliance

- Maintain all existing security policies
- Document compliance controls in Ansible
- Implement automated compliance checking

## Testing Strategy

1. **Unit Testing**: Test individual roles and modules
2. **Integration Testing**: Test interaction between roles
3. **System Testing**: Test complete playbooks in isolated environment
4. **Acceptance Testing**: Validate in staging environment
5. **Production Validation**: Carefully monitor initial production deployments

## Rollback Plan

1. Maintain existing Azure Automation DSC in parallel during migration
2. Document specific rollback procedures for each phase
3. Create snapshots/backups before major changes
4. Test rollback procedures in staging environment

## Documentation Requirements

1. Architecture documentation
2. Role and playbook documentation
3. Variable reference
4. Operational procedures
5. Troubleshooting guide

## Training Plan

1. Basic Ansible training for operations team
2. Role-specific training for specialized components
3. AWX/Tower administration training
4. Troubleshooting workshops

## Conclusion

This migration plan provides a comprehensive approach to convert the existing PowerShell DSC infrastructure to Ansible while maintaining all security features and functionality. The phased approach allows for careful testing and validation at each step, minimizing risk and ensuring a successful migration.

By leveraging Ansible's strengths in cross-platform management, improved readability, and extensive community support, the migrated infrastructure will be more maintainable and extensible while preserving the high security standards of the current implementation.